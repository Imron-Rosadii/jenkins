pipeline {
    agent any

    stages {
        stage('1. Clone Code') {
            steps {
                echo 'Mengunduh kode terbaru dari GitHub...'
                checkout scm
            }
        }
        
        stage('2. Test File') {
            steps {
                echo 'Memeriksa keberadaan file index.html...'
                // Jika Anda pakai Windows, ganti 'sh' menjadi 'bat'
                sh 'ls -la index.html' 
            }
        }
        
        stage('3. Build & Deploy Simulation') {
            steps {
                echo 'Simulasi Deployment: Aplikasi web statis siap digunakan!'
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline Sukses Terhubung dengan GitHub!'
        }
    }
}
