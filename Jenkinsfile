pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    stages {
        stage('1. Clone Code') {
            steps {
                echo "Branch yang diproses: ${env.BRANCH_NAME}"
                echo 'Mengunduh kode dari GitHub...'
                checkout scm
            }
        }

        stage('2. Validate HTML') {
            steps {
                echo 'Memeriksa file index.html...'
                sh 'test -s index.html'
                echo 'Validasi berhasil: index.html tersedia dan tidak kosong.'
            }
        }

        stage('3. Test') {
            steps {
                echo 'Menjalankan pengujian dasar...'
                sh 'test -r index.html'
                echo 'Pengujian dasar berhasil.'
            }
        }

        stage('4. Deploy to Staging (Simulation)') {
            when {
                branch 'staging'
            }
            steps {
                echo 'Simulasi deployment ke staging...'
                echo 'STAGING DEPLOYMENT SIMULATION SUCCESS'
            }
        }

        stage('5. Production Approval') {
            when {
                branch 'production'
            }
            steps {
                input(
                    message: 'Apakah kamu menyetujui deployment ke PRODUCTION?',
                    ok: 'Approve Deployment'
                )
            }
        }

        stage('6. Deploy to Production (Simulation)') {
            when {
                branch 'production'
            }
            steps {
                echo 'Persetujuan diterima.'
                echo 'Simulasi deployment ke production...'
                echo 'PRODUCTION DEPLOYMENT SIMULATION SUCCESS'
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
            echo 'PIPELINE ABORTED - Proses dihentikan atau dibatalkan.'
        }
    }
}