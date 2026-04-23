pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub')   // تأكد إن الـ ID هو 'dockerhub'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    sh 'docker build -t minac4/eshtry-mny-user:${IMAGE_TAG} ./User'
                    sh 'docker build -t minac4/eshtry-mny-product:${IMAGE_TAG} ./Product'
                    sh 'docker build -t minac4/eshtry-mny-cart:${IMAGE_TAG} ./Cart'
                    sh 'docker build -t minac4/eshtry-mny-frontend:${IMAGE_TAG} ./front-end'
                }
            }
        }

        // ==================== DevSecOps Stages ====================
        stage('Security: Dependency Audit') {
            steps {
                echo 'Running npm audit on all services...'
                sh 'cd User && npm audit || echo "User service has vulnerabilities"'
                sh 'cd Product && npm audit || echo "Product service has vulnerabilities"'
                sh 'cd Cart && npm audit || echo "Cart service has vulnerabilities"'
                sh 'cd front-end && npm audit || echo "Frontend has vulnerabilities"'
            }
        }

        stage('Security: Docker Scan (Trivy)') {
            steps {
                echo 'Scanning Docker images with Trivy (HIGH/CRITICAL only)...'
                sh '''
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-user:${IMAGE_TAG} --severity HIGH,CRITICAL --exit-code 0 || echo "User image has critical issues"
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-product:${IMAGE_TAG} --severity HIGH,CRITICAL --exit-code 0 || echo "Product image has critical issues"
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-cart:${IMAGE_TAG} --severity HIGH,CRITICAL --exit-code 0 || echo "Cart image has critical issues"
                    docker run --rm aquasec/trivy image minac4/eshtry-mny-frontend:${IMAGE_TAG} --severity HIGH,CRITICAL --exit-code 0 || echo "Frontend image has critical issues"
                '''
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                script {
                    sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                    sh 'docker push minac4/eshtry-mny-user:${IMAGE_TAG}'
                    sh 'docker push minac4/eshtry-mny-product:${IMAGE_TAG}'
                    sh 'docker push minac4/eshtry-mny-cart:${IMAGE_TAG}'
                    sh 'docker push minac4/eshtry-mny-frontend:${IMAGE_TAG}'
                }
            }
        }

        stage('Deploy to Kubernetes with Helm') {
            steps {
                script {
                    echo "Deploying to Minikube..."
                    sh '''
                        export KUBECONFIG=/var/lib/jenkins/.kube/config
                        echo "Checking Kubernetes connection..."
                        kubectl get nodes
                        cd eshtry-mny
                        helm upgrade --install eshtry-mny . \
                          --set images.user=minac4/eshtry-mny-user:${IMAGE_TAG} \
                          --set images.product=minac4/eshtry-mny-product:${IMAGE_TAG} \
                          --set images.cart=minac4/eshtry-mny-cart:${IMAGE_TAG} \
                          --set images.frontend=minac4/eshtry-mny-frontend:${IMAGE_TAG}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline Success - Application Deployed with Security Checks!'
        }
        failure {
            echo 'Pipeline Failed!'
        }
    }
}
