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
                        -e TERM=xterm \
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

        stage('Code Quality') {
            steps {
                withCredentials([string(credentialsId: 'SONAR_TOKEN', variable: 'SONAR_TOKEN')]) {
                    retry(2) {
                        sh '''
                            docker run --rm --volumes-from jenkins \
                                -e SONAR_TOKEN=${SONAR_TOKEN} \
                                sonarsource/sonar-scanner-cli \
                                -Dsonar.projectBaseDir=${WORKSPACE} \
                                -Dsonar.host.url=https://sonarcloud.io
                        '''
                    }
                }
            }
        }

        stage('Security') {
            steps {
                echo 'Scanning the built image for OS/library CVEs with Trivy...'
                sh '''
                    docker run --rm \
                        -v //var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy:latest image \
                            --scanners vuln \
                            --exit-code 0 --no-progress \
                            --severity HIGH,CRITICAL \
                            ${IMAGE_NAME}:${IMAGE_TAG}
                '''
                echo 'Running Brakeman static analysis (Rails SAST)...'
                sh '''
                    docker run --rm --volumes-from jenkins \
                        presidentbeef/brakeman \
                        --no-exit-on-warn --no-exit-on-error \
                        -p ${WORKSPACE}
                '''
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying to local staging with docker compose (port 3001)...'
                sh '''
                    docker compose -p df-staging -f docker-compose.staging.yml down -v --remove-orphans || true
                    docker compose -p df-staging -f docker-compose.staging.yml up -d

                    echo "Waiting for the staging API to respond on port 3001..."
                    for i in $(seq 1 60); do
                        if curl -fsS -H "Host: localhost" http://host.docker.internal:3001/api/docs/ >/dev/null 2>&1; then                            
                            echo "Staging API is live on http://localhost:3001"
                            break
                        fi
                        if [ "$i" = "60" ]; then
                            echo "Staging API did not come up in time. Recent logs:"
                            docker compose -p df-staging -f docker-compose.staging.yml logs --tail=60 staging-api
                            exit 1
                        fi
                        sleep 5
                    done
                '''
            }
        }
    }

    post {
        success { echo 'Pipeline completed successfully.' }
        failure { echo 'Pipeline failed.' }
    }
}