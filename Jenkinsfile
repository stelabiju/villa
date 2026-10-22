
pipeline {
    agent any

    environment {
        IMAGE_NAME = "stelabiju/villa-devsecops"
        SOURCE_REPO = "https://github.com/stelabiju/villa.git"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: "${SOURCE_REPO}"
            }
        }

        stage('SAST - Semgrep') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE:/src" \
                      semgrep/semgrep \
                      semgrep scan --config auto /src
                '''
            }
        }

        stage('Secret Scan - TruffleHog') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE:/src" \
                      trufflesecurity/trufflehog:latest \
                      filesystem /src --no-update
                '''
            }
        }

        stage('Dependency Check') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE:/src" \
                      -w /src \
                      python:3.12-slim \
                      sh -c "pip install --no-cache-dir pip-audit &&
                             pip-audit -r requirements.txt"
                '''
            }
        }

        stage('Django Check') {
            steps {
                sh '''
                    docker build -t villa-test .
                    docker run --rm \
                      -e DJANGO_SECRET_KEY=test-only-key \
                      -e DJANGO_ALLOWED_HOSTS=localhost \
                      villa-test \
                      python manage.py check
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-cred',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login \
                          -u "$DOCKER_USER" --password-stdin

                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
                        docker push ${IMAGE_NAME}:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Update GitOps Manifest') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'Github_jenkins',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_PASS'
                )]) {
                    sh '''
                        git config user.name "Jenkins"
                        git config user.email "jenkins@example.com"

                        sed -i "s|image: .*villa-devsecops:.*|image: ${IMAGE_NAME}:${BUILD_NUMBER}|" manifest/deployment.yaml

                        git add manifest/deployment.yaml

                        if git diff --cached --quiet; then
                            echo "No manifest changes to commit"
                        else
                            git commit -m "Update Villa image to build ${BUILD_NUMBER} [skip ci]"
                            git remote set-url origin https://${GIT_USER}:${GIT_PASS}@github.com/stelabiju/villa.git
                            git push origin HEAD:main
                        fi
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker image prune -f || true'
            echo 'Villa pipeline completed.'
        }
    }
}
