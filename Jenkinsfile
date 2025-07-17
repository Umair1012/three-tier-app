pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ECR_REPO_FRONTEND = 'three-tier-app-frontend'
        AWS_ECR_REPO_BACKEND = 'three-tier-app-backend'
        ECR_URI = '750067408510.dkr.ecr.us-east-1.amazonaws.com'

        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials'
        GHCR_CREDENTIALS_ID = 'ghcr'

        BRANCH_NAME = "${env.BRANCH_NAME}"
        IMAGE_TAG = "${BRANCH_NAME}-${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Login to Registries') {
            steps {
                // AWS ECR Login using env exports
                withCredentials([usernamePassword(credentialsId: 'aws-ecr',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URI}
                    '''
                }

                // Docker Hub Login with single credentials ID
                withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_PASS')]) {
                    sh '''
                        echo $DOCKERHUB_PASS | docker login -u $DOCKERHUB_USER --password-stdin
                    '''
                }

                // GHCR Login
                withCredentials([usernamePassword(credentialsId: "${GHCR_CREDENTIALS_ID}",
                    usernameVariable: 'GHCR_USER',
                    passwordVariable: 'GHCR_PAT')]) {
                    sh '''
                        echo $GHCR_PAT | docker login ghcr.io -u $GHCR_USER --password-stdin
                    '''
                }
            }
        }

        stage('Build & Tag Images') {
            steps {
                script {
                    env.FRONTEND_TAG_ECR = "${ECR_URI}/${AWS_ECR_REPO_FRONTEND}:${IMAGE_TAG}"
                    env.BACKEND_TAG_ECR = "${ECR_URI}/${AWS_ECR_REPO_BACKEND}:${IMAGE_TAG}"

                    env.FRONTEND_TAG_DH = "${DOCKERHUB_USER}/three-tier-app-frontend:${IMAGE_TAG}"
                    env.BACKEND_TAG_DH = "${DOCKERHUB_USER}/three-tier-app-backend:${IMAGE_TAG}"

                    env.FRONTEND_TAG_GHCR = "ghcr.io/${GHCR_USER}/three-tier-app-frontend:${IMAGE_TAG}"
                    env.BACKEND_TAG_GHCR = "ghcr.io/${GHCR_USER}/three-tier-app-backend:${IMAGE_TAG}"

                    sh """
                        docker build -t ${BACKEND_TAG_ECR} -t ${BACKEND_TAG_DH} -t ${BACKEND_TAG_GHCR} ./backend
                        docker build -t ${FRONTEND_TAG_ECR} -t ${FRONTEND_TAG_DH} -t ${FRONTEND_TAG_GHCR} ./frontend
                    """
                }
            }
        }

        stage('Push Images to All Registries') {
            steps {
                sh """
                    docker push ${BACKEND_TAG_ECR}
                    docker push ${FRONTEND_TAG_ECR}

                    docker push ${BACKEND_TAG_DH}
                    docker push ${FRONTEND_TAG_DH}

                    docker push ${BACKEND_TAG_GHCR}
                    docker push ${FRONTEND_TAG_GHCR}
                """
            }
        }

        stage('Prepare .env for Compose') {
            steps {
                script {
                    def envFile = ".env"
                    writeFile file: envFile, text: """
                    BACKEND_IMAGE=${BACKEND_TAG_ECR}
                    FRONTEND_IMAGE=${FRONTEND_TAG_ECR}
                    """
                }
            }
        }

        stage('Deploy Environment') {
            steps {
                script {
                    def composeFile = (BRANCH_NAME == "dev") ? "docker-compose.dev.yml" :
                                      (BRANCH_NAME == "stg") ? "docker-compose.stg.yml" : "docker-compose.prod.yml"

                    sh """
                        docker-compose --env-file .env -f ${composeFile} pull
                        docker-compose --env-file .env -f ${composeFile} down
                        docker-compose --env-file .env -f ${composeFile} up -d --remove-orphans
                    """
                }
            }
        }

        stage('Cleanup Local Images') {
            steps {
                sh """
                    docker rmi ${BACKEND_TAG_ECR} ${FRONTEND_TAG_ECR} || true
                    docker rmi ${BACKEND_TAG_DH} ${FRONTEND_TAG_DH} || true
                    docker rmi ${BACKEND_TAG_GHCR} ${FRONTEND_TAG_GHCR} || true
                """
            }
        }
    }

    post {
        success {
            echo "✅ ${BRANCH_NAME} environment deployed successfully with images pushed to AWS ECR, Docker Hub, and GHCR!"
        }
        failure {
            echo "❌ Deployment failed for ${BRANCH_NAME}. Check logs."
        }
    }
}



