pipeline {
    agent any

    environment {
        AWS_REGION = "us-east-1"
        ECR_REPO = "482421580240.dkr.ecr.us-east-1.amazonaws.com/amazon-clone"
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Clone Repo') {
            steps {
                git 'https://github.com/Priyanka101193Sharma/online-clone-amazon'
            }
        }

        stage('SonarQube Scan') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('sonar-server') {
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=amazon-clone \
                        -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t amazon-clone:${IMAGE_TAG} .
                """
            }
        }

        stage('Trivy Scan') {
            steps {
                sh """
                trivy image --exit-code 1 --severity CRITICAL amazon-clone:${IMAGE_TAG}
                """
            }
        }

        stage('Login to ECR') {
            steps {
                sh """
                aws ecr get-login-password --region ${AWS_REGION} | \
                docker login --username AWS --password-stdin ${ECR_REPO}
                """
            }
        }

        stage('Push to ECR') {
            steps {
                sh """
                docker tag amazon-clone:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}
                docker push ${ECR_REPO}:${IMAGE_TAG}
                """
            }
        }

        stage('Deploy to Kubernetes using Helm') {
            steps {
                sh """
                cd helm-chart
                helm upgrade --install amazon-release . \
                --set image.repository=${ECR_REPO} \
                --set image.tag=${IMAGE_TAG}
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh "kubectl rollout status deployment amazon-deployment"
            }
        }
    }

    post {
        success {
            echo "✅ Deployment Successful - Version ${IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline Failed - Rolling back..."
            sh "helm rollback amazon-release"
        }
    }
}
