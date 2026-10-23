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
        DEPLOYMENT_ATTEMPTED = 'false'
        DEPLOYMENT_VERIFIED  = 'false'
        HAD_PREVIOUS         = '0'
        PREVIOUS_CONTAINER   = ''
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

                    docker build \
                      -t "${IMAGE_NAME}:${BUILD_NUMBER}" \
                      -t "${IMAGE_NAME}:${GIT_COMMIT_SHORT}" \
                      .

                    docker image inspect \
                      "${IMAGE_NAME}:${GIT_COMMIT_SHORT}" >/dev/null

                    echo "Image build SUCCESS"
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
                      "${IMAGE_NAME}:${GIT_COMMIT_SHORT}"

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
                        script: '''
                            docker inspect "$APP_NAME" \
                              >/dev/null 2>&1 && echo 1 || echo 0
                        ''',
                        returnStdout: true
                    ).trim()

                    // Mulai sekarang, kegagalan pipeline perlu diperiksa
                    // oleh mekanisme rollback di bagian post.
                    env.DEPLOYMENT_ATTEMPTED = 'true'
                }

                sh '''
                    set -eu

                    OLD="$PREVIOUS_CONTAINER"
                    NEW_IMAGE="${IMAGE_NAME}:${GIT_COMMIT_SHORT}"

                    if [ "$HAD_PREVIOUS" = "1" ]; then
                        docker stop "$APP_NAME"
                        docker rename "$APP_NAME" "$OLD"
                    fi

                    docker run -d \
                      --name "$APP_NAME" \
                      --restart unless-stopped \
                      -p "${APP_PORT}:80" \
                      "$NEW_IMAGE"

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

                    # Periksa HTTP dari dalam container.
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
                        echo "Container health check FAILED"
                        docker logs "$APP_NAME" || true
                        exit 1
                    fi

                    # Periksa port produksi dari host melalui gateway
                    # jaringan Docker bridge.
                    HOST_GATEWAY=$(docker network inspect bridge \
                      --format '{{(index .IPAM.Config 0).Gateway}}')

                    OK=0
                    for i in $(seq 1 15); do
                        if wget -q -O /dev/null \
                          "http://${HOST_GATEWAY}:${APP_PORT}/"; then
                            OK=1
                            break
                        fi
                        sleep 2
                    done

                    if [ "$OK" -ne 1 ]; then
                        echo "Production endpoint check FAILED"
                        exit 1
                    fi

                    echo "Production verification SUCCESS"
                    docker port "$APP_NAME"

                    docker exec "$APP_NAME" \
                      wget -q -O - http://127.0.0.1/
                '''

                script {
                    env.DEPLOYMENT_VERIFIED = 'true'
                }
            }
        }
    }

    post {
        success {
           
            echo 'PIPELINE SUCCESS'
        }

        failure {
            script {
                if (env.BRANCH_NAME == 'production' &&
                    env.DEPLOYMENT_ATTEMPTED == 'true' &&
                    env.DEPLOYMENT_VERIFIED != 'true') {

                    echo 'Deployment failed. Attempting automatic rollback.'

                    sh '''
                        set -u

                        OLD="$PREVIOUS_CONTAINER"

                        if [ "$HAD_PREVIOUS" = "1" ]; then
                            if docker inspect "$OLD" >/dev/null 2>&1; then
                                # Container baru mungkin masih ada.
                                docker rm -f "$APP_NAME" \
                                  2>/dev/null || true

                                if docker rename "$OLD" "$APP_NAME" &&
                                   docker start "$APP_NAME"; then
                                    echo "Previous container restored."

                                    if docker exec "$APP_NAME" \
                                      wget -q -O /dev/null \
                                      http://127.0.0.1/; then
                                        echo "ROLLBACK SUCCESS"
                                    else
                                        echo "WARNING: Restored container failed health check."
                                        exit 1
                                    fi
                                else
                                    echo "CRITICAL: Automatic rollback failed."
                                    exit 1
                                fi
                            elif docker inspect "$APP_NAME" \
                                 >/dev/null 2>&1; then
                                # Container lama mungkin belum sempat
                                # diganti namanya.
                                docker start "$APP_NAME" || true

                                if docker exec "$APP_NAME" \
                                  wget -q -O /dev/null \
                                  http://127.0.0.1/; then
                                    echo "Existing container recovered."
                                else
                                    echo "CRITICAL: Existing container is unhealthy."
                                    exit 1
                                fi
                            else
                                echo "CRITICAL: Previous container not found."
                                exit 1
                            fi
                        else
                            # Tidak ada versi sebelumnya untuk dipulihkan.
                            docker rm -f "$APP_NAME" \
                              2>/dev/null || true
                            echo "No previous release existed."
                        fi
                    '''
                }
            }

            echo 'PIPELINE FAILURE - Check Console Output.'
        }

        aborted {
            echo 'PIPELINE ABORTED. Inspect deployment state before retrying.'
        }
    }
}