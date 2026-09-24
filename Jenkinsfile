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

        stage('Test') {
            environment {
                DB_CONTAINER    = "df-test-db-${env.BUILD_NUMBER}"
                REDIS_CONTAINER = "df-test-redis-${env.BUILD_NUMBER}"
                TEST_NETWORK    = "df-test-net-${env.BUILD_NUMBER}"
            }
            steps {
                sh '''
                    set -e

                    echo "Creating isolated test network and services..."
                    docker network create ${TEST_NETWORK}

                    docker run -d --name ${DB_CONTAINER} --network ${TEST_NETWORK} \
                        -e MYSQL_ROOT_PASSWORD=db-root-password \
                        -e MYSQL_DATABASE=doubtfire-test \
                        -e MYSQL_USER=dfire \
                        -e MYSQL_PASSWORD=pwd \
                        mariadb

                    docker run -d --name ${REDIS_CONTAINER} --network ${TEST_NETWORK} redis:7

                    echo "Waiting for the test database to accept connections..."
                    for i in $(seq 1 40); do
                        if docker exec ${DB_CONTAINER} sh -c 'mariadb-admin ping -uroot -pdb-root-password --silent 2>/dev/null || mysqladmin ping -uroot -pdb-root-password --silent 2>/dev/null'; then
                            echo "Database is ready."
                            break
                        fi
                        if [ "$i" = "40" ]; then
                            echo "Database did not become ready in time."; exit 1
                        fi
                        sleep 3
                    done

                    echo "Running the test suite inside the built image..."
                    docker run --rm --network ${TEST_NETWORK} \
                        -e RAILS_ENV=test \
                        -e CI=true \
                        -e DF_TEST_DB_ADAPTER=mysql2 \
                        -e DF_TEST_DB_HOST=${DB_CONTAINER} \
                        -e DF_TEST_DB_DATABASE=doubtfire-test \
                        -e DF_TEST_DB_USERNAME=dfire \
                        -e DF_TEST_DB_PASSWORD=pwd \
                        -e DF_REDIS_CACHE_URL=redis://${REDIS_CONTAINER}:6379/0 \
                        -e DF_REDIS_SIDEKIQ_URL=redis://${REDIS_CONTAINER}:6379/1 \
                        ${IMAGE_NAME}:${IMAGE_TAG} \
                        bash -c "bundle exec rails db:environment:set RAILS_ENV=test && RAILS_ENV=test bundle exec rake db:populate && bundle exec rails test"
                '''
            }
            post {
                always {
                    sh '''
                        echo "Cleaning up test services..."
                        docker rm -f ${DB_CONTAINER} ${REDIS_CONTAINER} || true
                        docker network rm ${TEST_NETWORK} || true
                    '''
                }
            }
        }
    }

    post {
        success { echo 'Pipeline completed successfully.' }
        failure { echo 'Pipeline failed.' }
    }
}