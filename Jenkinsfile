pipeline {
    agent any
    environment {
        AWS_REGION      = 'us-east-1'
        IMAGE_NAME      = 'nextflix'
        ECR_REGISTRY    = 'public.ecr.aws/s2g7y4g3'
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_BUILD_NUMBER = "${env.BUILD_NUMBER}"
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/zachary013/nextflix'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=nextflix \
                    -Dsonar.projectKey=nextflix'''
                }
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'OWASP-DP-CHECK'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                script {
                    try {
                        sh "trivy fs . > trivyfs.txt" 
                    }catch(Exception e){
                        input(message: "Are you sure to proceed?", ok: "Proceed")
                    }
                }
            }
        }
        stage("Docker Build Image"){
            steps{
                   
                sh """
                docker build --build-arg API_KEY=4361ca02a1df2d33722c1f4194a9aa42 -t ${IMAGE_NAME}:${DOCKER_BUILD_NUMBER} .
                """
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image netflix > trivyimage.txt"
                script{
                    input(message: "Are you sure to proceed?", ok: "Proceed")
                }
            }
        }
        
        stage("Docker Push to AWS ECR") {
            steps {
                script {
                    withAWS(region: "${AWS_REGION}", credentials: 'aws-credentials') {  

                        sh "aws ecr-public get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"

                        sh "docker tag ${IMAGE_NAME}:${DOCKER_BUILD_NUMBER} ${ECR_REGISTRY}/${IMAGE_NAME}:latest"

                        sh "docker push ${ECR_REGISTRY}/${IMAGE_NAME}:latest"
                    }
                }
            }
        }

    }
}