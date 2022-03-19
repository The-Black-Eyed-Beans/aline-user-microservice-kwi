pipeline {
    agent any

    tools {
        maven "Maven"
    }

    environment {
        AWS_ID = credentials("AWS-ACCOUNT-ID")
        REGION = credentials("REGION-KWI")
        PROJECT = "user-microservice"
        COMMIT_HASH = "${sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()}"
    }

    stages {
        stage("Test") {
            steps {
                echo "Initializing Git Submodule..."
                sh "git submodule init"
                sh "git submodule update"

                echo "Running Maven Test..."
                sh "mvn clean test"
            }
        }

        stage("SonarQube Code Analysis") {
            steps {
                echo "Running SonarQube Analysis..."
                withSonarQubeEnv(installationName: 'SonarQube-Server-kwi'){
                    sh "mvn verify sonar:sonar -Dsonar.projectName=${PROJECT}-kwi"
                }
            }
        }

        stage("Quality Gate"){
            steps {
                echo "Waiting for Quality Gate..."
                timeout(time: 5, unit: "MINUTES"){
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Package")  {
            steps {
                echo "Packaging Maven project..."
                sh "mvn package -DskipTests"
            }
        }

        stage("Docker Build") {
            steps {
                echo "Building Docker Image..."
                sh "aws ecr get-login-password --region ${REGION} --profile keshaun | sudo docker login --username AWS --password-stdin ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com"
                sh "sudo docker build -t ${PROJECT}-kwi:${COMMIT_HASH} ."

                echo "Uploading Docker Image to ECR..."
                sh "sudo docker tag ${PROJECT}-kwi:${COMMIT_HASH} ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"
                sh "sudo docker push ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"
            }
        }

        stage("Deployment") {
            steps {
                echo "Deploying ${PROJECT}-kwi..."
                sh '''
                aws cloudformation deploy \
                --stack-name ${PROJECT}-kwi-stack \
                --template-file deploy.json \
                --profile keshaun \
                --capabilities CAPABILITY_IAM \
                --no-fail-on-empty-changeset \
                --parameter-overrides \
                    MicroserviceName=${PROJECT} \
                    AppPort=8070 \
                    ImageTag=${COMMIT_HASH}
                '''
            }
        }
    }

    post {
        always {
            sh "sudo docker image rm ${PROJECT}-kwi:${COMMIT_HASH}"
            sh "sudo docker image rm ${AWS_ID}.dkr.ecr.${REGION}.amazonaws.com/${PROJECT}-kwi:${COMMIT_HASH}"
            sh "mvn clean"
        }
    }
}