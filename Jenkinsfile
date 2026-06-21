pipeline {
    agent any

    environment {
        IMAGE_NAME = "leyevge/hello-world"
        IMAGE_TAG = "v3"
        DOCKERHUB_CREDENTIALS = "dockerhub-credentials"
        GITHUB_GITOPS_CREDENTIALS = "github-gitops-credentials"
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
                    docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    trivy image \
                      --severity CRITICAL \
                      --exit-code 1 \
                      --no-progress \
                      ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('', DOCKERHUB_CREDENTIALS) {
                        docker.image("${IMAGE_NAME}:${IMAGE_TAG}").push()
                    }
                }
            }
        }

        stage('Update GitOps Configuration') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: GITHUB_GITOPS_CREDENTIALS,
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        git config user.email "jenkins@local"
                        git config user.name "Jenkins CI"

                        sed -i "s/tag: v[0-9]*/tag: ${IMAGE_TAG}/" environments/dev/values.yaml

                        if git diff --quiet; then
                            echo "GitOps configuration already uses ${IMAGE_TAG}"
                        else
                            git add environments/dev/values.yaml
                            git commit -m "Deploy dev image ${IMAGE_TAG} [skip ci]"
                            git push "https://${GIT_USERNAME}:${GIT_TOKEN}@github.com/yevgenylevin/hello-world-gitops.git" HEAD:main
                        fi
                    '''
                }
            }
        }
    }
}