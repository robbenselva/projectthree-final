pipeline {
    agent any
    
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKERHUB_REPO = 'robbenselvaone/project2-dev'
        IMAGE_TAG = "${BUILD_NUMBER}"
        AWS_REGION = 'ap-south-1'
        EKS_CLUSTER_NAME = 'jenkinsproject'
        KUBECONFIG = '/tmp/kubeconfig'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image..."
                    sh """
                        docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
                        docker tag ${DOCKERHUB_REPO}:${IMAGE_TAG} ${DOCKERHUB_REPO}:latest
                    """
                }
            }
        }
        
        stage('Push to DockerHub') {
            steps {
                script {
                    echo "Pushing image to DockerHub..."
                    sh """
                        echo \$DOCKERHUB_CREDENTIALS_PSW | docker login -u \$DOCKERHUB_CREDENTIALS_USR --password-stdin
                        docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                        docker push ${DOCKERHUB_REPO}:latest
                    """
                }
            }
        }
        
        stage('Update Kubernetes Deployment') {
            steps {
                script {
                    echo "Updating deployment.yml with new image..."
                    sh """
                        sed -i 's|image:.*|image: ${DOCKERHUB_REPO}:${IMAGE_TAG}|g' deployment.yml
                    """
                }
            }
        }
        
        stage('Configure kubectl for EKS') {
            steps {
                script {
                    echo "Configuring kubectl for EKS cluster..."
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                        sh """
                            aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME} --kubeconfig ${KUBECONFIG}
                            export KUBECONFIG=${KUBECONFIG}
                        """
                    }
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                script {
                    echo "Deploying to EKS cluster..."
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-credentials']]) {
                        sh """
                            export KUBECONFIG=${KUBECONFIG}
                            kubectl apply -f deployment.yml
                            kubectl rollout status deployment/your-deployment-name -n default
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh 'docker logout'
            cleanWs()
        }
        success {
            echo "Pipeline executed successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
