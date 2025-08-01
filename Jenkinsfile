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
                        # Java 17 works perfectly with WSO2 MI 4.4.0
                        export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
                        export PATH=$JAVA_HOME/bin:$PATH
                        
                        # Optimized Maven settings for 4.4.0
                        export MAVEN_OPTS="-Xms512m -Xmx2048m -XX:+UseG1GC -Djava.awt.headless=true"
                        
                        echo "Using Java for WSO2 MI 4.4.0:"
                        java -version
                        echo "Maven opts: $MAVEN_OPTS"
        
                        chmod +x ./mvnw
                        ./mvnw clean package -DskipTests
                        
                        echo "Build completed successfully!"
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

        stage('Complete Cleanup') {
            steps {
                script {
                    sh '''
                        echo "Performing complete cleanup of old deployments..."
                        microk8s kubectl delete deployment wso2mi-banking-document-system -n banking-document-system --ignore-not-found=true
                        microk8s kubectl delete pods --all -n banking-document-system --force --grace-period=0 --ignore-not-found=true
                        
                        echo "Waiting for cleanup to complete..."
                        sleep 15
                        
                        echo "Verifying cleanup..."
                        microk8s kubectl get pods -n banking-document-system || echo "No pods found - cleanup successful"
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh '''
                        echo "Applying Kubernetes manifests..."
                        microk8s kubectl apply -f k8s-manifests/namespace.yaml
                        microk8s kubectl apply -f k8s-manifests/configmap.yaml
                        microk8s kubectl apply -f k8s-manifests/service.yaml
                        microk8s kubectl apply -f k8s-manifests/deployment.yaml
                        microk8s kubectl apply -f k8s-manifests/ingress.yaml

                        echo "Waiting for deployment to complete..."
                        microk8s kubectl rollout status deployment/wso2mi-banking-document-system -n banking-document-system --timeout=900s
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
                        
                        echo "Checking application health..."
                        sleep 30
                        
                        # Get pod name and check if service is responding
                        POD_NAME=$(microk8s kubectl get pods -l app=wso2mi-banking-document-system -n banking-document-system -o jsonpath='{.items[0].metadata.name}')
                        echo "Testing application health on pod: $POD_NAME"
                        
                        # Test the health endpoint
                        microk8s kubectl exec -n banking-document-system $POD_NAME -- curl -f http://localhost:8290/services || echo "Service check failed - this might be normal during startup"
                    '''
                }
            }
        }

        stage('Final Health Check') {
            steps {
                script {
                    sh '''
                        echo "=== FINAL DEPLOYMENT STATUS ==="
                        
                        # Check pod status
                        echo "Pod Status:"
                        microk8s kubectl get pods -l app=wso2mi-banking-document-system -n banking-document-system -o wide
                        
                        # Check service endpoints
                        echo "Service Status:"
                        microk8s kubectl get services -n banking-document-system
                        
                        echo "Ingress Status:"
                        microk8s kubectl get ingress -n banking-document-system
                        
                        # Get recent logs
                        echo "Recent Application Logs:"
                        microk8s kubectl logs -l app=wso2mi-banking-document-system -n banking-document-system --tail=20 || echo "Could not retrieve logs"
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                sh '''
                    echo "=== POST-BUILD CLEANUP AND DIAGNOSTICS ==="
                    
                    # Always show current state
                    echo "Current Pod Status:"
                    microk8s kubectl get pods -l app=wso2mi-banking-document-system -n banking-document-system || true
                    
                    echo "Recent Events:"
                    microk8s kubectl get events -n banking-document-system --sort-by='.lastTimestamp' | tail -10 || true
                    
                    echo "Cleaning up unused Docker images..."
                    docker image prune -f
                '''
            }
        }

        success {
            echo '✅ Deployment successful!'
            script {
                sh '''
                    echo "=== DEPLOYMENT SUCCESS SUMMARY ==="
                    echo "Application endpoints:"
                    echo "Internal: http://wso2mi-banking-document-system-service.banking-document-system.svc.cluster.local:8290"
                    echo "External: http://banking-docs.local (add to /etc/hosts if using ingress)"
                    echo ""
                    echo "Available APIs (when fully started):"
                    echo "1. OTDS Authentication: POST /banking/otds/auth/credentials"
                    echo "2. Document Search: GET /banking/search/statements"
                    echo "3. Health Check: GET /services"
                    echo ""
                    echo "To check application status:"
                    echo "microk8s kubectl get pods -n banking-document-system"
                    echo "microk8s kubectl logs -l app=wso2mi-banking-document-system -n banking-document-system"
                '''
            }
        }

        failure {
            echo '❌ Deployment failed!'
            script {
                sh '''
                    echo "=== DEPLOYMENT FAILURE DIAGNOSTICS ==="
                    
                    echo "Pod Status:"
                    microk8s kubectl get pods -l app=wso2mi-banking-document-system -n banking-document-system -o wide || true
                    echo ""
                    
                    echo "Pod Descriptions:"
                    microk8s kubectl describe pods -l app=wso2mi-banking-document-system -n banking-document-system || true
                    echo ""
                    
                    echo "Container Logs (last 100 lines):"
                    microk8s kubectl logs -l app=wso2mi-banking-document-system -n banking-document-system --tail=100 || true
                    echo ""
                    
                    echo "Recent Events:"
                    microk8s kubectl get events -n banking-document-system --sort-by='.lastTimestamp' | tail -20 || true
                    echo ""
                    
                    echo "Deployment Status:"
                    microk8s kubectl get deployment wso2mi-banking-document-system -n banking-document-system -o yaml || true
                    echo ""
                    
                    echo "=== TROUBLESHOOTING TIPS ==="
                    echo "1. Check if the Docker image was built successfully"
                    echo "2. Verify Kubernetes resources are applied correctly"
                    echo "3. Check for resource constraints (CPU/Memory)"
                    echo "4. Verify ConfigMaps and Secrets are properly mounted"
                    echo "5. Check Java startup logs for JVM issues"
                '''
            }
        }

        unstable {
            echo '⚠️ Deployment completed with warnings!'
            script {
                sh '''
                    echo "=== DEPLOYMENT WARNINGS ==="
                    echo "Deployment completed but may have issues. Check logs above."
                    
                    echo "Current Status:"
                    microk8s kubectl get pods -l app=wso2mi-banking-document-system -n banking-document-system || true
                '''
            }
        }
    }
}
