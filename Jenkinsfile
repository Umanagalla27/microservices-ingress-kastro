pipeline {
    agent any

    environment {
        // ✅ Use your correct DockerHub repo name (no spaces, all lowercase)
        DOCKER_HUB_REPO = 'umanagalla27/techsolutions-app'
        K8S_CLUSTER_NAME = 'kastro-cluster'
        AWS_REGION = 'us-east-1'
        NAMESPACE = 'default'
        APP_NAME = 'techsolutions'
    }

    stages {
        // -------------------- CHECKOUT CODE --------------------
        stage('Checkout') {
            steps {
                echo '📦 Checking out source code...'
                git 'https://github.com/Umanagalla27/microservices-ingress-kastro.git'
            }
        }

        // -------------------- BUILD DOCKER IMAGE --------------------
        stage('Build Docker Image') {
            steps {
                echo '🔨 Building Docker image...'
                script {
                    def buildNumber = env.BUILD_NUMBER
                    def imageTag = "${DOCKER_HUB_REPO}:${buildNumber}"
                    def latestTag = "${DOCKER_HUB_REPO}:latest"

                    // Build image
                    sh "docker build -t ${imageTag} ."
                    sh "docker tag ${imageTag} ${latestTag}"

                    env.IMAGE_TAG = buildNumber
                }
            }
        }

        // -------------------- PUSH TO DOCKERHUB --------------------
        stage('Push to DockerHub') {
            steps {
                echo '📤 Pushing Docker image to DockerHub...'
                script {
                    // DockerHub credentials (username/password or token)
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        passwordVariable: 'DOCKER_PASSWORD',
                        usernameVariable: 'DOCKER_USERNAME'
                    )]) {
                        // Login to DockerHub
                        sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'

                        // Push both versioned & latest tags
                        sh "docker push ${DOCKER_HUB_REPO}:${env.IMAGE_TAG}"
                        sh "docker push ${DOCKER_HUB_REPO}:latest"
                    }
                }
            }
        }

        // -------------------- CONFIGURE AWS + KUBECTL --------------------
        stage('Configure AWS and Kubectl') {
            steps {
                echo '⚙️ Configuring AWS CLI and kubectl...'
                script {
                    // AWS Credentials binding
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                        sh "aws configure set region ${AWS_REGION}"
                        sh "aws eks update-kubeconfig --region ${AWS_REGION} --name ${K8S_CLUSTER_NAME}"
                        sh "kubectl get nodes"
                    }
                }
            }
        }

        // -------------------- DEPLOY TO KUBERNETES --------------------
        stage('Deploy to Kubernetes') {
            steps {
                echo '🚀 Deploying application to Kubernetes...'
                script {
                    // Replace Docker image tag dynamically
                    sh "sed -i 's|umanagalla27/techsolutions-app:latest|umanagalla27/techsolutions-app:${env.IMAGE_TAG}|g' k8s/deployment.yaml"

                    // Apply Kubernetes manifests
                    sh "kubectl apply -f k8s/deployment.yaml"
                    sh "kubectl apply -f k8s/service.yaml"

                    // Wait for rollout to complete
                    sh "kubectl rollout status deployment/${APP_NAME}-deployment --timeout=300s"
                    sh "kubectl get pods -l app=${APP_NAME}"
                }
            }
        }

        // -------------------- DEPLOY INGRESS --------------------
        stage('Deploy Ingress') {
            steps {
                echo '🌐 Deploying Ingress resource...'
                script {
                    sh "kubectl apply -f k8s/ingress.yaml"
                    sleep(10)
                    sh "kubectl get ingress ${APP_NAME}-ingress"
                }
            }
        }

        // -------------------- GET INGRESS URL --------------------
        stage('Get Ingress URL') {
            steps {
                echo '🔗 Fetching Ingress URL...'
                script {
                    timeout(time: 10, unit: 'MINUTES') {
                        waitUntil {
                            def result = sh(
                                script: "kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'",
                                returnStdout: true
                            ).trim()
                            if (result) {
                                env.INGRESS_URL = "http://${result}"
                                echo "Ingress URL: ${env.INGRESS_URL}"
                                return true
                            }
                            return false
                        }
                    }

                    echo "========================================="
                    echo "✅ DEPLOYMENT SUCCESSFUL!"
                    echo "========================================="
                    echo "Application URL: ${env.INGRESS_URL}"
                    echo ""
                    echo "Available Paths:"
                    echo "- Home Page: ${env.INGRESS_URL}/"
                    echo "- About Page: ${env.INGRESS_URL}/about"
                    echo "- Services Page: ${env.INGRESS_URL}/services"
                    echo "- Contact Page: ${env.INGRESS_URL}/contact"
                    echo "========================================="

                    // Health checks
                    sh "curl -I ${env.INGRESS_URL}/ || echo 'Home page check failed'"
                    sh "curl -I ${env.INGRESS_URL}/about || echo 'About page check failed'"
                    sh "curl -I ${env.INGRESS_URL}/services || echo 'Services page check failed'"
                    sh "curl -I ${env.INGRESS_URL}/contact || echo 'Contact page check failed'"
                }
            }
        }
    }

    // -------------------- POST STAGES --------------------
    post {
        always {
            echo '🧹 Cleaning up Docker images...'
            sh "docker rmi ${DOCKER_HUB_REPO}:${env.IMAGE_TAG} || true"
            sh "docker rmi ${DOCKER_HUB_REPO}:latest || true"
        }

        success {
            echo '🎉 Pipeline completed successfully!'
            echo "Access your application at: ${env.INGRESS_URL}"
        }

        failure {
            echo '❌ Pipeline failed! Please check the logs for details.'
        }
    }
}
