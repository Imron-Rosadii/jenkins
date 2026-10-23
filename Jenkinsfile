pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    environment {
        APP_NAME = 'portfolio-web'
        IMAGE_NAME = 'portfolio-web'
        APP_PORT = '8081'
        TEST_PORT = '8082'
    }

    stages {
        stage('1. Clone Code') {
            steps {
                echo "Branch yang diproses: ${env.BRANCH_NAME}"
                checkout scm
            }
        }

        stage('2. Validate HTML and Dockerfile') {
            steps {
                sh '''
                    set -eu
                    test -s index.html
                    test -s Dockerfile
                    echo "HTML dan Dockerfile tersedia."
                '''
            }
        }

        stage('3. Test') {
            steps {
                sh '''
                    set -eu
                    test -r index.html
                    grep -qi '<html' index.html
                    grep -qi 'FROM nginx' Dockerfile
                    echo "Validasi dasar berhasil."
                '''
            }
        }

        stage('4. Production Approval') {
            when {
                branch 'production'
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input(
                        message: 'Setujui build dan deployment aplikasi ke PRODUCTION?',
                        ok: 'Approve Deployment',
                        submitter: 'adminweb'
                    )
                }
            }
        }

        stage('5. Build Docker Image') {
            when {
                branch 'production'
            }
            steps {
                sh '''
                    set -eu
                    docker build \
                      --pull \
                      -t "${IMAGE_NAME}:${BUILD_NUMBER}" \
                      .
                    docker image inspect "${IMAGE_NAME}:${BUILD_NUMBER}" \
                      >/dev/null
                    echo "Docker image berhasil dibuat."
                '''
            }
        }

        stage('6. Test Candidate') {
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

                    # Tunggu Nginx siap merespons.
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

                    echo "Candidate berhasil melayani HTTP."
                    docker port "$CANDIDATE"
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

        stage('7. Deploy to Production') {
            when {
                branch 'production'
            }
            steps {
                sh '''
                    set -eu

                    OLD="${APP_NAME}-previous-${BUILD_NUMBER}"
                    HAD_OLD=0

                    # Simpan container aktif agar dapat di-rollback.
                    if docker inspect "$APP_NAME" >/dev/null 2>&1; then
                        docker stop "$APP_NAME"
                        docker rename "$APP_NAME" "$OLD"
                        HAD_OLD=1
                    fi

                    # Jalankan versi baru pada port production.
                    if ! docker run -d \
                      --name "$APP_NAME" \
                      --restart unless-stopped \
                      -p "${APP_PORT}:80" \
                      "${IMAGE_NAME}:${BUILD_NUMBER}"; then

                        if [ "$HAD_OLD" -eq 1 ]; then
                            docker rename "$OLD" "$APP_NAME"
                            docker start "$APP_NAME"
                        fi
                        exit 1
                    fi

                    # Verifikasi aplikasi di dalam container baru.
                    if ! docker exec "$APP_NAME" \
                      wget -q -O /dev/null http://127.0.0.1/; then

                        docker logs "$APP_NAME" || true
                        docker rm -f "$APP_NAME" || true

                        if [ "$HAD_OLD" -eq 1 ]; then
                            docker rename "$OLD" "$APP_NAME"
                            docker start "$APP_NAME"
                        fi
                        exit 1
                    fi

                    echo "Deployment production berhasil."
                '''
            }
        }

        stage('8. Verify Production') {
            when {
                branch 'production'
            }
            steps {
                sh '''
                    set -eu
                    docker ps --filter "name=^/${APP_NAME}$"
                    docker port "$APP_NAME"
                    docker exec "$APP_NAME" \
                      wget -q -O - http://127.0.0.1/
                    echo
                    echo "PRODUCTION VERIFICATION SUCCESS"
                '''
            }
        }
    }

    post {
        success {
            echo 'PIPELINE SUCCESS'
        }
        failure {
            echo 'PIPELINE FAILURE - Periksa Console Output.'
        }
        aborted {
            echo 'PIPELINE ABORTED.'
        }
    }
}