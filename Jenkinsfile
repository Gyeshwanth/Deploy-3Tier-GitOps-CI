
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
      GIT_USER_NAME  = "Gyeshwanth"
        GIT_REPO_NAME  = "Deploy-3Tier-GitOps-CD"
	}

    stages {
        stage('git checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'git-cred',
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
            withCredentials([usernamePassword(credentialsId: 'git-cred', usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD')]) {
                sh '''
                    echo "🌀 Cloning CD repo..."
                    rm -rf cd
                    git clone https://$GIT_USERNAME:$GIT_PASSWORD@github.com/$GIT_USER_NAME/$GIT_REPO_NAME.git cd
                    cd cd
                    git checkout main || git checkout -b main

                    echo " Installing yq..."
                    curl -sL https://github.com/mikefarah/yq/releases/download/v4.44.3/yq_linux_amd64 -o yq
                    chmod +x yq
                    mv yq /tmp/yq
                    export PATH=$PATH:/tmp

                    echo " Updating image tags..."
                    echo "📝 Updating backend image tag..."
                    yq -i '
                      (. | select(.kind=="Deployment")
                         | .spec.template.spec.containers[]
                         | select(.name=="backend")
                         | .image)
                      = "yeshwanthgosi/backend:" + strenv(IMAGE_TAG)
                    ' k8s-prod/backend.yaml

                    yq -i '
                      (. | select(.kind=="Deployment")
                         | .spec.template.spec.containers[]
                         | select(.name=="frontend")
                         | .image)
                      = "yeshwanthgosi/frontend:" + strenv(IMAGE_TAG)
                    ' k8s-prod/frontend.yaml

                    git config user.email "yeshwanth@example.com"
                    git config user.name "yeshwanth"
                    git add -A
                    git commit -m "Update image tags to ${IMAGE_TAG}" || echo "No changes to commit."
                    git pull --rebase origin main || true

                    echo " Pushing updated files..."
                    git push https://$GIT_USERNAME:$GIT_PASSWORD@github.com/$GIT_USER_NAME/$GIT_REPO_NAME.git HEAD:main || echo "⚠️ Push failed or no new changes."
                '''
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
