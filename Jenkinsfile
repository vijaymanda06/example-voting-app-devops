pipeline {
    agent { label 'docker-agent' }

    environment {
        AWS_REGION = "ap-south-1"
        AWS_ACCOUNT_ID = "823151423293"
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/vijaymanda06/example-voting-app-devops.git'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                scannerHome = tool 'sonar-scanner'
            }
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                    $scannerHome/bin/sonar-scanner \
                      -Dsonar.projectKey=voting-app \
                      -Dsonar.sources=. \
                      -Dsonar.host.url=http://65.0.132.145:9000
                    '''
                }
            }
        }

        stage('Login to ECR') {
            steps {
                sh '''
                aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
                '''
            }
        }

        stage('Build and Push Images') {
            steps {
                sh '''
                # Vote
                docker build -t vote ./vote
                docker tag vote:latest $ECR_REGISTRY/vote:${BUILD_NUMBER}
                docker push $ECR_REGISTRY/vote:${BUILD_NUMBER}

                # Result
                docker build -t result ./result
                docker tag result:latest $ECR_REGISTRY/result:${BUILD_NUMBER}
                docker push $ECR_REGISTRY/result:${BUILD_NUMBER}

                # Worker
                docker build -t worker ./worker
                docker tag worker:latest $ECR_REGISTRY/worker:${BUILD_NUMBER}
                docker push $ECR_REGISTRY/worker:${BUILD_NUMBER}
                '''
            }
        }

        stage('Update Kubernetes Manifests') {
            steps {
                sh '''
                sed -i "s|image:.*vote.*|image: $ECR_REGISTRY/vote:${BUILD_NUMBER}|" k8s-specifications/vote-deployment.yaml
                sed -i "s|image:.*result.*|image: $ECR_REGISTRY/result:${BUILD_NUMBER}|" k8s-specifications/result-deployment.yaml
                sed -i "s|image:.*worker.*|image: $ECR_REGISTRY/worker:${BUILD_NUMBER}|" k8s-specifications/worker-deployment.yaml
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                aws eks --region $AWS_REGION update-kubeconfig --name voting-cluster
                kubectl apply -f k8s-specifications/
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline finished! Build #${BUILD_NUMBER}"
        }
    }
}
