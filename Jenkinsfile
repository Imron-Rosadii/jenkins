
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APP_NAME             = 'portfolio-web'
        IMAGE_NAME           = 'portfolio-web'
        APP_PORT             = '8081'
        TEST_PORT            = '8082'
        DEPLOYMENT_ATTEMPTED = 'false'
        DEPLOYMENT_VERIFIED  = 'false'
        HAD_PREVIOUS         = 'false'
        PREVIOUS_CONTAINER   = ''
        PREVIOUS_IMAGE_ID    = ''
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
                    env.DEPLOYMENT_VERIFIED = 'false'
                    def backupName = "${env.APP_NAME}-previous-${env.BUILD_NUMBER}"

                    def backupExists = sh(
                        script: "docker inspect '${backupName}' >/dev/null 2>&1",
                        returnStatus: true
                    ) == 0

                    if (backupExists) {
                        error("Backup container already exists: ${backupName}")
                    }

                    env.PREVIOUS_CONTAINER = backupName

                    env.HAD_PREVIOUS = sh(
                        script: '''
                            if docker inspect "$APP_NAME" >/dev/null 2>&1; then
                                echo true
                            else
                                echo false
                            fi
                        ''',
                        returnStdout: true
                    ).trim()

                    env.PREVIOUS_IMAGE_ID = ''

                    if (env.HAD_PREVIOUS == 'true') {
                        env.PREVIOUS_IMAGE_ID = sh(
                            script: '''
                                docker inspect --format '{{.Image}}' "$APP_NAME"
                            ''',
                            returnStdout: true
                        ).trim()
                    }

                    env.DEPLOYMENT_ATTEMPTED = 'true'

                    echo "Previous release exists: ${env.HAD_PREVIOUS}"
                    echo "Backup name: ${env.PREVIOUS_CONTAINER}"
                    echo "Previous image ID: ${env.PREVIOUS_IMAGE_ID}"

                    sh '''
                        set -eu

                        OLD="$PREVIOUS_CONTAINER"
                        NEW_IMAGE="${IMAGE_NAME}:${GIT_COMMIT_SHORT}"

                        if [ "$HAD_PREVIOUS" = "true" ]; then
                            # Cegah backup lama otomatis restart.
                            docker update --restart=no "$APP_NAME"
                            docker stop "$APP_NAME"
                            docker rename "$APP_NAME" "$OLD"

                            echo "Previous release saved as $OLD"
                        else
                            echo "No previous release exists."
                        fi

                        docker run -d \
                            --name "$APP_NAME" \
                            --restart unless-stopped \
                            -p "${APP_PORT}:80" \
                            "$NEW_IMAGE"

                        echo "New production container started: $NEW_IMAGE"
                    '''
                    }
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

        unsuccessful {
            script {
                if (env.BRANCH_NAME == 'production' &&
                    env.DEPLOYMENT_ATTEMPTED == 'true' &&
                    env.DEPLOYMENT_VERIFIED != 'true') {

                    echo 'Deployment unsuccessful. Starting rollback recovery.'

                    sh '''
                        set -eu

                        OLD="$PREVIOUS_CONTAINER"

                        if docker inspect "$OLD" >/dev/null 2>&1; then
                            BACKUP_IMAGE_ID=$(docker inspect \
                             --format '{{.Image}}' "$OLD")

                            if [ "$BACKUP_IMAGE_ID" != "$PREVIOUS_IMAGE_ID" ]; then
                                echo "CRITICAL: Backup image does not match previous release."
                                exit 1
                            fi
                            # Backup ditemukan: pulihkan release lama.
                            echo "Previous release found: $OLD"

                            docker rm -f "$APP_NAME" 2>/dev/null || true

                            docker rename "$OLD" "$APP_NAME"
                            docker update --restart=unless-stopped "$APP_NAME"
                            docker start "$APP_NAME"

                            echo "Previous release restored."

                        elif [ "$HAD_PREVIOUS" = "true" ]; then
                            if ! docker inspect "$APP_NAME" >/dev/null 2>&1; then
                                echo "CRITICAL: Previous container is missing."
                                exit 1
                            fi

                            CURRENT_IMAGE_ID=$(docker inspect \
                                --format '{{.Image}}' "$APP_NAME")

                            if [ "$CURRENT_IMAGE_ID" != "$PREVIOUS_IMAGE_ID" ]; then
                                echo "CRITICAL: Current container is not the previous release."
                                exit 1
                            fi

                            echo "Previous container is still under its original name."

                            docker update --restart=unless-stopped "$APP_NAME"

                            if [ "$(docker inspect \
                                -f '{{.State.Running}}' "$APP_NAME")" != "true" ]; then
                                docker start "$APP_NAME"
                            fi

                        else
                            # First deployment: tidak ada release lama.
                            echo "No previous release was available."

                            if docker inspect "$APP_NAME" >/dev/null 2>&1; then
                                docker rm -f "$APP_NAME"
                            fi

                            echo "Failed first deployment cleaned up."
                            exit 0
                        fi

                        # Verifikasi container yang dipulihkan.
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

                        # Verifikasi endpoint port production.
                        # Gunakan image release sebelumnya sebagai pemeriksa endpoint.
                        OK=0

                        for i in $(seq 1 15); do
                            if docker run --rm \
                                --network host \
                                --entrypoint wget \
                                "$PREVIOUS_IMAGE_ID" \
                                -q -O /dev/null \
                                "http://127.0.0.1:${APP_PORT}/"; then
                                OK=1
                                break
                            fi

                            sleep 2
                        done

                        if [ "$OK" -ne 1 ]; then
                            echo "CRITICAL: Restored endpoint check FAILED."
                            exit 1
                        fi

                        echo "Restored production endpoint check PASSED."

                        echo "Previous production release is healthy."
                    '''
                }
            }
        }

        failure {
            echo 'PIPELINE FAILURE - Check Console Output.'
        }

        aborted {
            echo 'PIPELINE ABORTED - Inspect deployment state.'
        }
    }

}
