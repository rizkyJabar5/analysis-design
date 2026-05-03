pipeline {
    agent any

    environment {
        BACKEND_API_URL = 'https://343e-103-145-224-199.ngrok-free.app/api/v1/analyze' 
        // API_TOKEN = credentials('backend-api-token') 
    }

    // Mengatur agar Jenkins tidak otomatis gagal jika proses memakan waktu agak lama
    options {
        timeout(time: 15, unit: 'MINUTES') 
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
                    
                    // 1. Mencoba mencari dari Git History
                    def files = sh(
                        script: "git diff-tree --no-commit-id --name-only -r HEAD | grep -E '\\.(xml|puml|uml|jpg|jpeg|png)\$' || true",
                        returnStdout: true
                    ).trim()

                    // 2. FALLBACK: Jika Git diff kosong (sering terjadi di build pertama Jenkins), 
                    // cari SEMUA file desain di repositori agar API tetap berjalan.
                    if (files == "") {
                        echo "⚠️ Git diff tidak mendeteksi perubahan (Mungkin build pertama atau shallow clone)."
                        echo "🔍 FALLBACK: Memindai seluruh file desain di repositori..."
                        files = sh(
                            script: "find . -type f \\( -name '*.xml' -o -name '*.puml' -o -name '*.uml' -o -name '*.jpg' -o -name '*.png' \\) | sed 's|^./||' || true",
                            returnStdout: true
                        ).trim()
                    }

                    env.CHANGED_FILES = files

                    if (env.CHANGED_FILES == "") {
                        echo "Tidak ada satupun file desain di dalam repositori ini. Pipeline dilewati."
                    } else {
                        echo "File desain yang akan dikirim ke Backend:\n${env.CHANGED_FILES}"
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
                    def fileArray = env.CHANGED_FILES.split('\n')
                    def pipelineFailed = false

                    // Menyiapkan header untuk file HTML
                    sh """
                    echo '<!DOCTYPE html><html><head><title>Laporan Keamanan Desain</title>' > summary.html
                    echo '<style>body{font-family:"Segoe UI",Tahoma,sans-serif; padding:20px; color:#333; line-height:1.6;} .header{background:#f8f9fa; padding:20px; border-radius:8px; margin-bottom:20px;} .threat-card{border:1px solid #ddd; padding:15px; margin-bottom:15px; border-radius:8px;} .CRITICAL{border-left:6px solid #dc3545; background:#fff5f5;} .HIGH{border-left:6px solid #fd7e14; background:#fff9f4;} .MEDIUM{border-left:6px solid #ffc107; background:#fffdf5;} .LOW{border-left:6px solid #17a2b8;} .btn{display:inline-block; padding:10px 15px; background:#007bff; color:white; text-decoration:none; border-radius:5px; font-weight:bold; margin-top:10px;}</style>' >> summary.html
                    echo '</head><body>' >> summary.html
                    """

                    for (int i = 0; i < fileArray.size(); i++) {
                        def file = fileArray[i].trim()
                        if (file == "") continue

                        echo "------------------------------------------------"
                        echo "🚀 Mengirim file ke Backend: ${file}"
                        echo "⏳ Menunggu proses analisa dari AI (Mohon bersabar, ini memakan waktu)..."

                        // Hit API Backend menggunakan cURL
                        // Ditambahkan timeout spesifik di curl agar tidak hang selamanya jika ngrok bermasalah
                        sh """
                        curl -s -S -w "\\n%{http_code}" --max-time 600 -X POST ${BACKEND_API_URL} \
                             -F "design_file=@${file}" > raw_response.txt
                        """

                        // Ekstrak HTTP Status dan Body
                        def responseLines = readFile('raw_response.txt').split('\n')
                        def httpStatus = responseLines[-1].trim()
                        
                        def jsonBody = responseLines[0..-2].join('\n')
                        writeFile file: 'response.json', text: jsonBody

                        if (httpStatus != "200") {
                            echo "❌ ERROR: Backend mengembalikan status HTTP ${httpStatus}"
                            echo "Isi Response: ${jsonBody}"
                            pipelineFailed = true
                            continue
                        }

                        // Parse JSON
                        def isPassed = sh(script: "jq -r '.passedQualityGate' response.json", returnStdout: true).trim()
                        def reportPdf = sh(script: "jq -r '.downloadLinks.pdf' response.json", returnStdout: true).trim()
                        def securityScore = sh(script: "jq -r '.score' response.json", returnStdout: true).trim()
                        def totalThreats = sh(script: "jq '.threats | length' response.json", returnStdout: true).trim()

                        echo "📊 HASIL ANALISA: ${file}"
                        echo "Skor: ${securityScore}/100 | Quality Gate Passed: ${isPassed}"
                        
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

                        sh '''
                        jq -r '.threats[]? | "<div class=\\"threat-card \\(.severity)\\"><h4 style=\\"margin-top:0; color:#444;\\">🚨 \\(.componentName) - \\(.strideCategory) [\\(.severity)]</h4><p><strong>Temuan:</strong> \\(.description)</p><p><strong>Mitigasi Teknikal:</strong> <em>\\(.technicalMitigation)</em></p></div>"' response.json >> summary.html
                        '''

                        if (isPassed != "true") {
                            pipelineFailed = true
                        }
                    }

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

    post {
        always {
            echo "Mempublikasikan Visualisasi HTML..."
            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '', 
                reportFiles: 'summary.html',
                reportName: 'Security Threat Summary',
                reportTitles: 'Hasil Analisa Keamanan Desain'
            ])
            
            sh "rm -f raw_response.txt response.json"
        }
    }
}
