pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
        KUBECONFIG_CREDENTIAL = 'kubeconfig'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        DOCKERHUB_USER = 'taantiope'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo "Checked out branch: ${env.GIT_BRANCH}"
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build Movie Service') {
                    steps {
                        script {
                            echo "Building movie-service image..."
                            sh """
                                cd movie-service
                                docker build -t ${DOCKERHUB_USER}/movie-service:${IMAGE_TAG} .
                                docker tag ${DOCKERHUB_USER}/movie-service:${IMAGE_TAG} ${DOCKERHUB_USER}/movie-service:latest
                            """
                        }
                    }
                }
                stage('Build Cast Service') {
                    steps {
                        script {
                            echo "Building cast-service image..."
                            sh """
                                cd cast-service
                                docker build -t ${DOCKERHUB_USER}/cast-service:${IMAGE_TAG} .
                                docker tag ${DOCKERHUB_USER}/cast-service:${IMAGE_TAG} ${DOCKERHUB_USER}/cast-service:latest
                            """
                        }
                    }
                }
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    echo "Running basic image verification..."
                    sh """
                        # Verify images exist and get their info
                        echo "Verifying movie-service image..."
                        docker inspect ${DOCKERHUB_USER}/movie-service:${IMAGE_TAG} > /dev/null 2>&1
                        if [ \$? -eq 0 ]; then
                            echo "✓ movie-service:${IMAGE_TAG} image built successfully"
                            docker history ${DOCKERHUB_USER}/movie-service:${IMAGE_TAG} | head -5
                        else
                            echo "✗ Failed to build movie-service image"
                            exit 1
                        fi

                        echo ""
                        echo "Verifying cast-service image..."
                        docker inspect ${DOCKERHUB_USER}/cast-service:${IMAGE_TAG} > /dev/null 2>&1
                        if [ \$? -eq 0 ]; then
                            echo "✓ cast-service:${IMAGE_TAG} image built successfully"
                            docker history ${DOCKERHUB_USER}/cast-service:${IMAGE_TAG} | head -5
                        else
                            echo "✗ Failed to build cast-service image"
                            exit 1
                        fi

                        echo ""
                        echo "✅ All images verified successfully"
                    """
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                script {
                    echo "Pushing images to DockerHub..."
                    sh """
                        echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin
                        docker push ${DOCKERHUB_USER}/movie-service:${IMAGE_TAG}
                        docker push ${DOCKERHUB_USER}/movie-service:latest
                        docker push ${DOCKERHUB_USER}/cast-service:${IMAGE_TAG}
                        docker push ${DOCKERHUB_USER}/cast-service:latest
                        echo "✅ Images pushed successfully"
                    """
                }
            }
        }

        stage('Deploy to Dev') {
            when {
                branch 'develop'
            }
            steps {
                script {
                    echo "Deploying to Dev environment..."
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL}", variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}
                            kubectl config use-context docker-desktop
                            helm dependency update ./helm-charts/movieapp || true
                            helm upgrade --install movieapp ./helm-charts/movieapp \
                                --values ./helm-charts/movieapp/values-dev.yaml \
                                --namespace dev \
                                --set movieService.image.tag=${IMAGE_TAG} \
                                --set castService.image.tag=${IMAGE_TAG} \
                                --create-namespace \
                                --wait \
                                --timeout 5m
                            kubectl get pods -n dev
                        """
                    }
                    echo "✅ Deployed to Dev"
                }
            }
        }

        stage('Deploy to QA') {
            when {
                branch 'qa'
            }
            steps {
                script {
                    echo "Deploying to QA environment..."
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL}", variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}
                            kubectl config use-context docker-desktop
                            helm dependency update ./helm-charts/movieapp || true
                            helm upgrade --install movieapp ./helm-charts/movieapp \
                                --values ./helm-charts/movieapp/values-qa.yaml \
                                --namespace qa \
                                --set movieService.image.tag=${IMAGE_TAG} \
                                --set castService.image.tag=${IMAGE_TAG} \
                                --create-namespace \
                                --wait \
                                --timeout 5m
                            kubectl get pods -n qa
                        """
                    }
                    echo "✅ Deployed to QA"
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'staging'
            }
            steps {
                script {
                    echo "Deploying to Staging environment..."
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL}", variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}
                            kubectl config use-context docker-desktop
                            helm dependency update ./helm-charts/movieapp || true
                            helm upgrade --install movieapp ./helm-charts/movieapp \
                                --values ./helm-charts/movieapp/values-staging.yaml \
                                --namespace staging \
                                --set movieService.image.tag=${IMAGE_TAG} \
                                --set castService.image.tag=${IMAGE_TAG} \
                                --create-namespace \
                                --wait \
                                --timeout 5m
                            kubectl get pods -n staging
                        """
                    }
                    echo "✅ Deployed to Staging"
                }
            }
        }

        stage('Manual Approval for Production') {
            when {
                branch 'master'
            }
            steps {
                script {
                    echo "⚠️  PRODUCTION DEPLOYMENT REQUIRES APPROVAL"
                    input message: 'Deploy to Production?',
                          ok: 'Deploy to Production',
                          submitter: 'admin'
                }
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'master'
            }
            steps {
                script {
                    echo "Deploying to Production environment..."
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL}", variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}
                            kubectl config use-context docker-desktop
                            helm dependency update ./helm-charts/movieapp || true
                            helm upgrade --install movieapp ./helm-charts/movieapp \
                                --values ./helm-charts/movieapp/values-prod.yaml \
                                --namespace prod \
                                --set movieService.image.tag=${IMAGE_TAG} \
                                --set castService.image.tag=${IMAGE_TAG} \
                                --create-namespace \
                                --wait \
                                --timeout 5m
                            kubectl get pods -n prod
                        """
                    }
                    echo "✅ Deployed to Production"
                }
            }
        }

        stage('Post-Deployment Verification') {
            steps {
                script {
                    withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL}", variable: 'KUBECONFIG_FILE')]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG_FILE}

                            # Determine namespace based on branch
                            if [ "${env.GIT_BRANCH}" = "master" ]; then
                                NAMESPACE="prod"
                            elif [ "${env.GIT_BRANCH}" = "staging" ]; then
                                NAMESPACE="staging"
                            elif [ "${env.GIT_BRANCH}" = "qa" ]; then
                                NAMESPACE="qa"
                            elif [ "${env.GIT_BRANCH}" = "develop" ]; then
                                NAMESPACE="dev"
                            else
                                NAMESPACE="dev"
                            fi

                            echo "=== Deployment verification for namespace: \$NAMESPACE ==="
                            kubectl get pods -n \$NAMESPACE
                            kubectl get services -n \$NAMESPACE
                            echo "✅ Verification complete"
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline execution completed"
            sh 'docker system prune -f || true'
        }
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed. Check the logs for details."
        }
    }
}
