pipeline {
    agent any

    stages {
        stage('1. Clone Code') {
            steps {
                echo 'Mengunduh kode terbaru dari GitHub...'
                checkout scm
            }
        }
        
        stage('2. Validasi & Pengujian Kode (Test)') {
            steps {
                echo 'Memulai proses validasi dokumen...'
                
                // VALIDASI 1: Memeriksa apakah file index.html ada di dalam repository
                // Catatan: Jika Jenkins Anda di Windows, ganti 'sh' menjadi 'bat'
                sh 'ls -la index.html' 
                
                // VALIDASI 2: Memeriksa apakah file tersebut kosong atau tidak
                echo 'Memastikan file index.html berisi kode...'
                sh '[ -s index.html ]' // Akan error jika file berukuran 0 byte
                
                // VALIDASI 3: Simulasi pengecekan tag HTML (opsional)
                echo 'Validasi struktur HTML berhasil dilewati dengan sukses.'
            }
        }
        
        stage('3. Build & Deploy Simulation') {
            steps {
                echo 'Aplikasi lolos validasi!'
                echo 'Simulasi Deployment: Aplikasi web statis siap digunakan di server!'
            }
        }
    }
    
    post {
        success {
            echo '==================================================='
            echo ' Hasil: VALIDASI BERHASIL & PIPELINE SUKSES! '
            echo '==================================================='
        }
        failure {
            echo '==================================================='
            echo ' Hasil: VALIDASI GAGAL! Periksa kembali kode Anda. '
            echo '==================================================='
        }
    }
}
