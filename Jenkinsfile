pipeline {
    agent any

    tools {
        nodejs 'nodejs23'
    }

    environment {
        SCANNER_HOME   = tool 'sonar-scanner'
        IMAGE_TAG      = "1.0.${BUILD_NUMBER}"
        FRONTEND_IMAGE = "yeshwanthgosi/frontend"
        BACKEND_IMAGE  = "yeshwanthgosi/backend"
        CD_REPO_URL    = "https://github.com/Gyeshwanth/Deploy-3Tier-GitOps-CD.git"
    }

    stages {
        stage('git checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'gitcred',
                    url: 'https://github.com/Gyeshwanth/Deploy-3Tier-GitOps-CI.git'
            }
        }

        stage('frontend compile') {
            steps {
                dir('client') {
                    sh 'find . -name "*.js" -exec node --check {} +'
                }
            }
        }

        stage('backend compile') {
            steps {
                dir('api') {
                    sh 'find . -name "*.js" -exec node --check {} +'
                }
            }
        }

        stage('git leaks') {
            steps {
                sh 'gitleaks detect --source ./client --exit-code 1'
                sh 'gitleaks detect --source ./api --exit-code 1'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=NodeJS-Project \
                        -Dsonar.projectKey=NodeJS-Project '''
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o fs-report.html .'
            }
        }

       
        stage('Build, Tag & Push Backend Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        dir('api') {
                            sh """
                                echo " Building backend image..."
                                docker build -t ${BACKEND_IMAGE}:latest -t ${BACKEND_IMAGE}:${IMAGE_TAG} .
                                echo " Scanning image with Trivy..."
                                trivy image --format table -o backend-image-report.html ${BACKEND_IMAGE}:${IMAGE_TAG} || true
                                echo " Pushing images..."
                                docker push ${BACKEND_IMAGE}:latest
                                docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                            """
                        }
                    }
                }
            }
        }

        stage('Build, Tag & Push Frontend Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        dir('client') {
                            sh """
                                echo " Building frontend image..."
                                docker build -t ${FRONTEND_IMAGE}:latest -t ${FRONTEND_IMAGE}:${IMAGE_TAG} .
                                echo " Scanning image with Trivy..."
                                trivy image --format table -o frontend-image-report.html ${FRONTEND_IMAGE}:${IMAGE_TAG} || true
                                echo " Pushing images..."
                                docker push ${FRONTEND_IMAGE}:latest
                                docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                            """
                        }
                    }
                }
            }
        }

        stage('Update CD Repo with New Image Tags') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'gitcred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                        sh """
                            echo " Cloning CD repo..."
                            rm -rf cd
                            git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Gyeshwanth/Deploy-3Tier-GitOps-CD.git cd
                            cd cd
                            git checkout main || git checkout -b main

                            echo " Installing yq..."
                            curl -sL https://github.com/mikefarah/yq/releases/download/v4.44.3/yq_linux_amd64 -o yq
                            sudo install yq /usr/local/bin/yq

                            echo "️ Updating image tags..."
                            yq -i '
                              (. | select(.kind=="Deployment")
                               | .spec.template.spec.containers[] 
                               | select(.name=="backend") 
                               | .image) = "${BACKEND_IMAGE}:${IMAGE_TAG}"
                            ' k8s-prod/backend.yaml

                            yq -i '
                              (. | select(.kind=="Deployment")
                               | .spec.template.spec.containers[] 
                               | select(.name=="frontend") 
                               | .image) = "${FRONTEND_IMAGE}:${IMAGE_TAG}"
                            ' k8s-prod/frontend.yaml

                            echo " Configuring Git..."
                            git config user.email "yeshwanth@example.com"
                            git config user.name "yeshwanth"

                            echo " Committing and pushing changes..."
                            git add -A
                            git commit -m "Update image tags to ${IMAGE_TAG}" || echo "No changes to commit."
                            git pull --rebase origin main || true

                            echo " Pushing to GitHub main branch..."
                            GIT_USER_NAME="Gyeshwanth"
                            GIT_REPO_NAME="Deploy-3Tier-GitOps-CD"
                            git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git HEAD:main || echo "⚠️ Push failed or no new changes."
                        """
                    }
                }
            }
        }

    }

    post {
        success {
            echo "✅ CI + CD pipeline completed successfully. Images updated to tag: ${IMAGE_TAG}"
        }
        failure {
            echo "❌ Pipeline failed. Check Jenkins logs for details."
        }
    }
}
