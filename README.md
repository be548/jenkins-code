pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'docker.io'                // Docker registry, e.g., Docker Hub
        DOCKER_CREDENTIALS_ID = 'docker-credentials' // Jenkins credentials ID for Docker
        KUBE_CONFIG_CREDENTIALS_ID = 'kubeconfig-credentials' // Jenkins credentials ID for Kubernetes
        K8S_NAMESPACE = 'default'                   // Kubernetes namespace
    }

    parameters {
        string(name: 'SERVICE_NAME', defaultValue: 'my-microservice', description: 'Name of the microservice to deploy')
        string(name: 'DOCKER_IMAGE_TAG', defaultValue: 'latest', description: 'Docker image tag to use for deployment')
        string(name: 'DEPLOYMENT_NAME', defaultValue: 'my-microservice-deployment', description: 'Kubernetes Deployment name')
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out the microservice source code..."
                    // Checkout from version control
                    checkout scm
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker image for ${params.SERVICE_NAME}"
                    sh """
                        docker build -t ${DOCKER_REGISTRY}/${params.SERVICE_NAME}:${params.DOCKER_IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    echo "Pushing Docker image to ${DOCKER_REGISTRY}"
                    withCredentials([usernamePassword(credentialsId: DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin ${DOCKER_REGISTRY}
                            docker push ${DOCKER_REGISTRY}/${params.SERVICE_NAME}:${params.DOCKER_IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "Deploying ${params.SERVICE_NAME} to Kubernetes cluster"
                    withCredentials([file(credentialsId: KUBE_CONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG')]) {
                        sh """
                            kubectl --kubeconfig=$KUBECONFIG set image deployment/${params.DEPLOYMENT_NAME} ${params.SERVICE_NAME}=${DOCKER_REGISTRY}/${params.SERVICE_NAME}:${params.DOCKER_IMAGE_TAG} --namespace=${K8S_NAMESPACE}
                            kubectl --kubeconfig=$KUBECONFIG rollout status deployment/${params.DEPLOYMENT_NAME} --namespace=${K8S_NAMESPACE}
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Deployment successful!"
        }
        failure {
            echo "Deployment failed!"
        }
    }
}
