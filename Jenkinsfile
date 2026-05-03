pipeline {
    agent any

    environment {
        BACKEND_API_URL = 'https://343e-103-145-224-199.ngrok-free.app/api/v1/analyze' 
        
        // Token otorisasi (kosongkan nilainya jika backend belum pakai auth)
        // API_TOKEN = credentials('backend-api-token') 
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Mengambil desain terbaru dari repositori Git...'
                checkout scm
            }
        }

        stage('Validasi File Commit') {
            steps {
                script {
                    echo 'Mencari file desain yang baru saja di-commit...'
                    // Mengambil daftar file yang berubah
                    env.CHANGED_FILES = sh(
                        script: "git diff-tree --no-commit-id --name-only -r HEAD | grep -E '\\.(xml|puml|uml|jpg|jpeg|png)\$' || true",
                        returnStdout: true
                    ).trim()

                    if (env.CHANGED_FILES == "") {
                        echo "Tidak ada file desain yang relevan pada commit ini. Pipeline dilewati."
                        currentBuild.result = 'SUCCESS'
                        sh "exit 0" 
                    } else {
                        echo "File desain ditemukan:\n${env.CHANGED_FILES}"
                    }
                }
            }
        }

        stage('Analisa & Render Visual') {
            when {
                expression { env.CHANGED_FILES != "" }
            }
            steps {
                script {
                    def files = env.CHANGED_FILES.split('\n')
                    def pipelineFailed = false

                    // Menyiapkan header untuk file HTML
                    sh """
                    echo '<!DOCTYPE html><html><head><title>Laporan Keamanan Desain</title>' > summary.html
                    echo '<style>body{font-family:"Segoe UI",Tahoma,sans-serif; padding:20px; color:#333; line-height:1.6;} .header{background:#f8f9fa; padding:20px; border-radius:8px; margin-bottom:20px;} .threat-card{border:1px solid #ddd; padding:15px; margin-bottom:15px; border-radius:8px;} .CRITICAL{border-left:6px solid #dc3545; background:#fff5f5;} .HIGH{border-left:6px solid #fd7e14; background:#fff9f4;} .MEDIUM{border-left:6px solid #ffc107; background:#fffdf5;} .LOW{border-left:6px solid #17a2b8;} .btn{display:inline-block; padding:10px 15px; background:#007bff; color:white; text-decoration:none; border-radius:5px; font-weight:bold; margin-top:10px;}</style>' >> summary.html
                    echo '</head><body>' >> summary.html
                    """

                    for (int i = 0; i < files.size(); i++) {
                        def file = files[i]
                        echo "------------------------------------------------"
                        echo "🚀 Mengirim file ke Backend: ${file}"

                        // Hit API Backend menggunakan cURL dan simpan ke raw_response.txt
                        sh """
                        curl -s -w "\\n%{http_code}" -X POST ${BACKEND_API_URL} \
                             -F "design_file=@${file}" > raw_response.txt
                        """

                        // Ekstrak HTTP Status dan Body
                        def responseLines = readFile('raw_response.txt').split('\n')
                        def httpStatus = responseLines[-1].trim()
                        
                        // Gabungkan baris body dan simpan sebagai file JSON murni
                        def jsonBody = responseLines[0..-2].join('\n')
                        writeFile file: 'response.json', text: jsonBody

                        if (httpStatus != "200") {
                            echo "❌ ERROR: Backend mengembalikan status HTTP ${httpStatus}"
                            pipelineFailed = true
                            continue
                        }

                        // Parse JSON menggunakan jq membaca dari file response.json
                        def isPassed = sh(script: "jq -r '.passedQualityGate' response.json", returnStdout: true).trim()
                        def reportPdf = sh(script: "jq -r '.downloadLinks.pdf' response.json", returnStdout: true).trim()
                        def securityScore = sh(script: "jq -r '.score' response.json", returnStdout: true).trim()
                        def totalThreats = sh(script: "jq '.threats | length' response.json", returnStdout: true).trim()

                        // Cetak log di terminal Jenkins
                        echo "📊 HASIL ANALISA: ${file}"
                        echo "Skor: ${securityScore}/100 | Quality Gate Passed: ${isPassed}"
                        
                        // Menulis Header Laporan ke HTML
                        def statusColor = (isPassed == "true") ? "green" : "red"
                        def statusText = (isPassed == "true") ? "LOLOS (AMAN)" : "GAGAL (RENTAN)"
                        
                        sh """
                        echo '<div class="header">' >> summary.html
                        echo '<h2>🛡️ File Desain: ${file}</h2>' >> summary.html
                        echo '<p><strong>Skor Keamanan STRIDE:</strong> ${securityScore}/100</p>' >> summary.html
                        echo '<p><strong>Status Quality Gate:</strong> <span style="color: ${statusColor}; font-weight: bold;">${statusText}</span></p>' >> summary.html
                        echo '<p><strong>Total Ancaman:</strong> ${totalThreats}</p>' >> summary.html
                        echo '<a href="${reportPdf}" target="_blank" class="btn">📄 Unduh Laporan PDF Lengkap</a>' >> summary.html
                        echo '</div>' >> summary.html
                        echo '<h3>Detail Ancaman Berdasarkan Komponen:</h3>' >> summary.html
                        """

                        // Looping Threats menggunakan jq dan render HTML Card
                        sh '''
                        jq -r '.threats[] | "<div class=\\"threat-card \\(.severity)\\"><h4 style=\\"margin-top:0; color:#444;\\">🚨 \\(.componentName) - \\(.strideCategory) [\\(.severity)]</h4><p><strong>Temuan:</strong> \\(.description)</p><p><strong>Mitigasi Teknikal:</strong> <em>\\(.technicalMitigation)</em></p></div>"' response.json >> summary.html
                        '''

                        if (isPassed != "true") {
                            pipelineFailed = true
                        }
                    }

                    // Tutup tag HTML
                    sh "echo '</body></html>' >> summary.html"

                    if (pipelineFailed) {
                        error("🚨 Pipeline dihentikan! Quality Gate GAGAL karena terdapat kerentanan pada desain.")
                    } else {
                        echo "🎉 Semua desain berhasil lolos uji keamanan (Quality Gate PASSED)."
                    }
                }
            }
        }
    }

    // TAHAP POST-ACTION: Publikasi Visual
    post {
        always {
            echo "Mempublikasikan Visualisasi HTML..."
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '', // Kosong berarti di root workspace Jenkins
                reportFiles: 'summary.html',
                reportName: 'Security Threat Summary',
                reportTitles: 'Hasil Analisa Keamanan Desain'
            ])
            
            // Bersihkan file sementara
            sh "rm -f raw_response.txt response.json"
        }
    }
}
