@Library('sharedLib') _

def IMAGE_TAG = "${env.BRANCH_NAME}-${env.BUILD_NUMBER}"

def COMPOSE_FILE = "docker-compose.yml"

pipeline {
    agent any

    stages {
        stage('Checkout') { steps { checkoutSource() } }
        stage('Docker Login') { steps { dockerLogin('dockerhub-credentials') } }
        stage('Build Images') { steps { buildImages('./backend', './frontend', IMAGE_TAG) } }
        stage('Push Images') { steps { pushImages() } }
        stage('Prepare .env') { steps { prepareEnvFile() } }
        stage('Deploy') {
            steps {
                    deployApp(COMPOSE_FILE, env.BRANCH_NAME)
                }
            }
        }
        stage('Cleanup') { steps { cleanupImages() } }
    }
}

    post {
        success {
            echo "✅ ${BRANCH_NAME} environment deployed successfully using Docker Hub images!"
        }
        failure {
            echo "❌ Deployment failed for ${BRANCH_NAME}. Check logs."
        }
    }
}
