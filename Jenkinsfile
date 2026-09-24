pipeline {
    agent any

    environment {
        IMAGE_NAME = 'doubtfire-api'
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
    }

    options {
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Checked out commit ${env.GIT_COMMIT}"
            }
        }
        stage('Build') {
            steps {
                echo "Building ${IMAGE_NAME}:${IMAGE_TAG}"
                sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest .'
            }
        }
    }

    post {
        success { echo 'Pipeline completed successfully.' }
        failure { echo 'Pipeline failed.' }
    }
}