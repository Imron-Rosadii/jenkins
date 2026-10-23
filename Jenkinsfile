
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        APP_NAME   = 'portfolio-web'
        IMAGE_NAME = 'portfolio-web'
        APP_PORT   = '8081'
        TEST_PORT  = '8082'
    }

    stages {
        stage('1. Clone Code') {
            steps {
                echo "Branch: ${env.BRANCH_NAME}"
                checkout scm

                script {
                    env.GIT_COMMIT_SHORT = sh(
                        script: 'git rev-parse --short=12 HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Git commit: ${env.GIT_COMMIT_SHORT}"
                }
            }
        }

        stage('2. Validate') {
            steps {
                sh '''
                    set -eu
                    test -s index.html
                    test -s Dockerfile
                    grep -qi '<html' index.html
                    grep -qi 'FROM nginx' Dockerfile
                    echo "Validation SUCCESS"
                '''
            }
        }

        stage('3. Production Approval') {
            when {
                branch 'production'
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input(
                        message: 'Approve production deployment?',
                        ok: 'Approve Deployment',
                        submitter: 'adminweb'
                    )
                }
            }
        }

        stage('4. Build Docker Image') {
            when {
                branch 'production'
            }
            steps {
                sh '''
                    set -eu

                    docker build --pull \
                      -t "${IMAGE_NAME}:${BUILD_NUMBER}" \
                      -t "${IMAGE_NAME}:${GIT_COMMIT_SHORT}" \
                      .

                    docker image inspect \
                      "${IMAGE_NAME}:${BUILD_NUMBER}" >/dev/null

                    docker image inspect \
                      "${IMAGE_NAME}:${GIT_COMMIT_SHORT}" >/dev/null

                    echo "Image build SUCCESS"
                    echo "Build tag: ${IMAGE_NAME}:${BUILD_NUMBER}"
                    echo "Commit tag: ${IMAGE_NAME}:${GIT_COMMIT_SHORT}"
                '''
            }
        }

        stage('5. Test Candidate') {
            when {
                branch 'production'
            }
            steps {
                sh '''
                    set -eu

                    CANDIDATE="${APP_NAME}-candidate-${BUILD_NUMBER}"

                    docker rm -f "$CANDIDATE" 2>/dev/null || true

                    docker run -d \
                      --name "$CANDIDATE" \
                      -p "${TEST_PORT}:80" \
                      "${IMAGE_NAME}:${BUILD_NUMBER}"

                    OK=0

                    for i in $(seq 1 15); do
                        if docker exec "$CANDIDATE" \
                          wget -q -O /dev/null http://127.0.0.1/; then
                            OK=1
                            break
                        fi
                        sleep 2
                    done

                    if [ "$OK" -ne 1 ]; then
                        docker logs "$CANDIDATE" || true
                        exit 1
                    fi

                    echo "Candidate test SUCCESS"
                '''
            }
            post {
                always {
                    sh '''
                        docker rm -f \
                          "${APP_NAME}-candidate-${BUILD_NUMBER}" \
                          2>/dev/null || true
                    '''
                }
            }
        }

        stage('6. Deploy Production') {
            when {
                branch 'production'
            }
            steps {
                script {
                    env.PREVIOUS_CONTAINER =
                        "${env.APP_NAME}-previous-${env.BUILD_NUMBER}"

                    env.HAD_PREVIOUS = sh(
                        script: 'docker inspect portfolio-web >/dev/null 2>&1 && echo 1 || echo 0',
                        returnStdout: true
                    ).trim()
                }

                sh '''
                    set -eu

                    OLD="$PREVIOUS_CONTAINER"
                    HAD_OLD="$HAD_PREVIOUS"
                    NEW_IMAGE="${IMAGE_NAME}:${GIT_COMMIT_SHORT}"

                    # Simpan container lama untuk rollback.
                    if [ "$HAD_OLD" = "1" ]; then
                        docker stop "$APP_NAME"
                        docker rename "$APP_NAME" "$OLD"
                    fi

                    # Jalankan image baru.
                    if ! docker run -d \
                      --name "$APP_NAME" \
                      --restart unless-stopped \
                      -p "${APP_PORT}:80" \
                      "$NEW_IMAGE"; then

                        if [ "$HAD_OLD" = "1" ]; then
                            docker rename "$OLD" "$APP_NAME"
                            docker start "$APP_NAME"
                        fi

                        exit 1
                    fi

                    echo "New container started: $NEW_IMAGE"
                '''
            }
        }

        stage('7. Verify Production') {
            when {
                branch 'production'
            }
            steps {
                sh '''
                    set -eu

                    OK=0

                    for i in $(seq 1 15); do
                        if docker exec "$APP_NAME" \
                          wget -q -O /dev/null http://127.0.0.1/; then
                            OK=1
                            break
                        fi
                        sleep 2
                    done

                    if [ "$OK" -ne 1 ]; then
                        echo "Production verification FAILED"
                        docker logs "$APP_NAME" || true

                        # Hapus container versi baru.
                        docker rm -f "$APP_NAME" 2>/dev/null || true

                        # Pulihkan container sebelumnya.
                        if [ "$HAD_PREVIOUS" = "1" ]; then
                            docker rename "$PREVIOUS_CONTAINER" "$APP_NAME"
                            docker start "$APP_NAME"
                            echo "ROLLBACK SUCCESS"
                        else
                            echo "No previous container available."
                        fi

                        exit 1
                    fi

                    echo "Production verification SUCCESS"
                    docker ps --filter "name=^/${APP_NAME}$"
                    docker port "$APP_NAME"

                    docker exec "$APP_NAME" \
                      wget -q -O - http://127.0.0.1/

                    echo
                '''
            }
        }
    }

    post {
        success {
            echo 'PIPELINE SUCCESS'
        }
        failure {
            echo 'PIPELINE FAILURE - Check Console Output.'
        }
        aborted {
            echo 'PIPELINE ABORTED.'
        }
    }
}
