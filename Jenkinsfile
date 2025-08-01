pipeline {
    agent any

    environment {
        DOCKERHUB_REGISTRY = 'docker.io'
        DOCKERHUB_USERNAME = 'medlas'
        IMAGE_NAME = 'wso2mi-banking-document-system'
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
        NAMESPACE = 'banking-document-system'
        GIT_REPO = 'https://github.com/med-las/wso2.git'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: "${GIT_REPO}"
            }
        }

        stage('Build Maven') {
            steps {
                script {
                    sh '''
                        export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
                        export PATH=$JAVA_HOME/bin:$PATH
                        echo "Using Java:"
                        java -version

                        chmod +x ./mvnw
                        ./mvnw clean package -DskipTests

                        echo "Target directory:"
                        ls -la target/
                        echo "Checking for CAR file:"
                        ls -la target/*.car
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${BUILD_NUMBER}"
                    def imageName = "${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${imageTag}"
                    def latestImage = "${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest"

                    sh """
                        docker build -f deployment/docker/Dockerfile -t ${imageName} .
                        docker tag ${imageName} ${latestImage}
                    """

                    env.IMAGE_TAG = imageTag
                    env.FULL_IMAGE_NAME = imageName
                    env.LATEST_IMAGE_NAME = latestImage
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )]) {
                        sh '''
                            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                            docker push ${FULL_IMAGE_NAME}
                            docker push ${LATEST_IMAGE_NAME}
                        '''
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh '''
                        microk8s kubectl apply -f k8s-manifests/namespace.yaml
                        microk8s kubectl apply -f k8s-manifests/configmap.yaml
                        microk8s kubectl apply -f k8s-manifests/service.yaml
                        microk8s kubectl apply -f k8s-manifests/deployment.yaml
                        microk8s kubectl apply -f k8s-manifests/ingress.yaml

                        microk8s kubectl rollout status deployment/wso2mi-banking-document-system -n banking-document-system --timeout=600s
                    '''
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    sh '''
                        echo "Checking deployment status..."
                        microk8s kubectl get pods -n banking-document-system
                        microk8s kubectl get services -n banking-document-system

                        echo "Waiting for pods to be ready..."
                        microk8s kubectl wait --for=condition=ready pod -l app=wso2mi-banking-document-system -n banking-document-system --timeout=600s
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                sh '''
                    echo "Cleaning up unused Docker images..."
                    docker image prune -f
                '''
            }
        }

        success {
            echo '✅ Deployment successful!'
            script {
                sh '''
                    echo "Application endpoints:"
                    echo "Internal: http://wso2mi-banking-document-system-service.banking-document-system.svc.cluster.local:8290"
                    echo "External: http://banking-docs.local (add to /etc/hosts)"
                    echo ""
                    echo "Available APIs:"
                    echo "1. OTDS Authentication: POST /banking/otds/auth/credentials"
                    echo "2. Document Search: GET /banking/search/statements"
                '''
            }
        }

        failure {
            echo '❌ Deployment failed!'
            script {
                sh '''
                    echo "Deployment logs:"
                    microk8s kubectl logs -l app=wso2mi-banking-document-system -n banking-document-system --tail=50
                '''
            }
        }
    }
}
