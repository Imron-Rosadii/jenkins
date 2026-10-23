
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        APP_NAME             = 'portfolio-web'
        IMAGE_NAME           = 'portfolio-web'
        APP_PORT             = '8081'
        TEST_PORT            = '8082'
        DEPLOYMENT_ATTEMPTED = 'false'
        DEPLOYMENT_VERIFIED  = 'false'
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
                        echo "Candidate health check FAILED"
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
                    env.DEPLOYMENT_ATTEMPTED = 'true'
                }

                sh '''
                    set -eu

                    OLD="${APP_NAME}-previous-${BUILD_NUMBER}"
                    NEW_IMAGE="${IMAGE_NAME}:${GIT_COMMIT_SHORT}"

                    # Hindari benturan nama backup.
                    if docker inspect "$OLD" >/dev/null 2>&1; then
                        echo "ERROR: Backup container already exists: $OLD"
                        exit 1
                    fi

                    # Simpan release lama jika tersedia.
                    if docker inspect "$APP_NAME" >/dev/null 2>&1; then
                        docker stop "$APP_NAME"
                        docker rename "$APP_NAME" "$OLD"
                    fi

                    # Jalankan release baru.
                    if ! docker run -d \
                        --name "$APP_NAME" \
                        --restart unless-stopped \
                        -p "${APP_PORT}:80" \
                        "$NEW_IMAGE"; then

                        echo "ERROR: New container failed to start"

                        # Pulihkan release lama jika tersedia.
                        if docker inspect "$OLD" >/dev/null 2>&1; then
                            docker rm -f "$APP_NAME" 2>/dev/null || true
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

                    NEW_IMAGE="${IMAGE_NAME}:${GIT_COMMIT_SHORT}"

                    echo "Checking Nginx inside production container..."

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

                    echo "Container health check PASSED"
                    echo "Checking published host port ${APP_PORT}..."

                    # Jalankan pemeriksa sementara di network namespace host.
                    # Tidak membutuhkan curl/wget di container Jenkins.
                    OK=0

                    for i in $(seq 1 15); do
                        if docker run --rm \
                            --network host \
                            --entrypoint wget \
                            "$NEW_IMAGE" \
                            -q -O /dev/null \
                            "http://127.0.0.1:${APP_PORT}/"; then
                            OK=1
                            break
                        fi

                        sleep 2
                    done

                    if [ "$OK" -ne 1 ]; then
                        echo "Production endpoint check FAILED"
                        exit 1
                    fi

                    echo "Production endpoint check PASSED"
                    docker port "$APP_NAME"

                    echo "Production HTML:"
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
                        set -eu

                        OLD="${APP_NAME}-previous-${BUILD_NUMBER}"

                        if docker inspect "$OLD" >/dev/null 2>&1; then
                            # Hentikan/hapus release baru jika ada.
                            docker rm -f "$APP_NAME" 2>/dev/null || true

                            # Pulihkan release sebelumnya.
                            docker rename "$OLD" "$APP_NAME"
                            docker start "$APP_NAME"

                            echo "Previous container restored"

                        elif docker inspect "$APP_NAME" >/dev/null 2>&1; then
                            # Tidak ada backup bernama OLD.
                            # Jangan mengklaim rollback berhasil.
                            echo "WARNING: No backup container found."
                            echo "Current container has been left in place."
                            exit 1

                        else
                            echo "CRITICAL: No production container available."
                            exit 1
                        fi

                        # Verifikasi release yang dipulihkan.
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
                            echo "CRITICAL: Rollback health check FAILED."
                            docker logs "$APP_NAME" || true
                            exit 1
                        fi

                        echo "ROLLBACK SUCCESS"
                    '''
                }
            }

            echo 'PIPELINE FAILURE - Check Console Output.'
        }

        aborted {
            echo 'PIPELINE ABORTED'
        }
    }
}
