pipeline {
    agent any

    environment {
        BACKEND_API_URL = 'http://172.235.251.236:8080/api/v1/analyze' 
        // API_TOKEN = credentials('backend-api-token') 
    }

    // Prevent Jenkins from failing automatically if the process takes a while
    options {
        timeout(time: 15, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Fetching the latest design from the Git repository...'
                checkout scm
            }
        }

        stage('Validate Commit Files') {
            steps {
                script {
                    echo 'Looking for recently committed design files...'

                    // 1. Try to find them from Git history
                    def files = sh(
                        script: "git diff-tree --no-commit-id --name-only -r HEAD | grep -E '\\.(xml|puml|uml|jpg|jpeg|png)\$' || true",
                        returnStdout: true
                    ).trim()

                    // 2. FALLBACK: If the Git diff is empty (often happens on Jenkins' first build),
                    // scan ALL design files in the repository so the API still runs.
                    if (files == "") {
                        echo "⚠️ Git diff did not detect any changes (Possibly first build or shallow clone)."
                        echo "🔍 FALLBACK: Scanning all design files in the repository..."
                        files = sh(
                            script: "find . -type f \\( -name '*.xml' -o -name '*.puml' -o -name '*.uml' -o -name '*.jpg' -o -name '*.png' \\) | sed 's|^./||' || true",
                            returnStdout: true
                        ).trim()
                    }

                    env.CHANGED_FILES = files

                    if (env.CHANGED_FILES == "") {
                        echo "No design files exist in this repository. Skipping pipeline."
                    } else {
                        echo "Design files to be sent to the Backend:\n${env.CHANGED_FILES}"
                    }
                }
            }
        }

        stage('Analyze & Render Visual') {
            when {
                expression { env.CHANGED_FILES != "" }
            }
            steps {
                script {
                    def fileArray = env.CHANGED_FILES.split('\n')
                    def pipelineFailed = false

                    // Prepare the header for the HTML file
                    sh """
                    echo '<!DOCTYPE html><html><head><title>Design Security Report</title>' > summary.html
                    echo '<style>body{font-family:"Segoe UI",Tahoma,sans-serif; padding:20px; color:#333; line-height:1.6;} .header{background:#f8f9fa; padding:20px; border-radius:8px; margin-bottom:20px;} .threat-card{border:1px solid #ddd; padding:15px; margin-bottom:15px; border-radius:8px;} .CRITICAL{border-left:6px solid #dc3545; background:#fff5f5;} .HIGH{border-left:6px solid #fd7e14; background:#fff9f4;} .MEDIUM{border-left:6px solid #ffc107; background:#fffdf5;} .LOW{border-left:6px solid #17a2b8;} .btn{display:inline-block; padding:10px 15px; background:#007bff; color:white; text-decoration:none; border-radius:5px; font-weight:bold; margin-top:10px;}</style>' >> summary.html
                    echo '</head><body>' >> summary.html
                    """

                    for (int i = 0; i < fileArray.size(); i++) {
                        def file = fileArray[i].trim()
                        if (file == "") continue

                        echo "------------------------------------------------"
                        echo "🚀 Sending file to the Backend: ${file}"
                        echo "⏳ Waiting for the AI analysis process (Please be patient, this takes time)..."

                        // Hit the Backend API using cURL
                        // A specific curl timeout is added so it doesn't hang forever if ngrok has issues
                        sh """
                        curl -s -S -w "\\n%{http_code}" --max-time 600 -X POST ${BACKEND_API_URL} \
                             -F "design_file=@${file}" > raw_response.txt
                        """

                        // Extract HTTP Status and Body
                        def responseLines = readFile('raw_response.txt').split('\n')
                        def httpStatus = responseLines[-1].trim()

                        def jsonBody = responseLines[0..-2].join('\n')
                        writeFile file: 'response.json', text: jsonBody

                        if (httpStatus != "200") {
                            echo "❌ ERROR: Backend returned HTTP status ${httpStatus}"
                            echo "Response body: ${jsonBody}"
                            pipelineFailed = true
                            continue
                        }

                        // Parse JSON
                        def isPassed = sh(script: "jq -r '.passedQualityGate' response.json", returnStdout: true).trim()
                        def reportPdf = sh(script: "jq -r '.downloadLinks.pdf' response.json", returnStdout: true).trim()
                        def securityScore = sh(script: "jq -r '.score' response.json", returnStdout: true).trim()
                        def totalThreats = sh(script: "jq '.threats | length' response.json", returnStdout: true).trim()

                        echo "📊 ANALYSIS RESULT: ${file}"
                        echo "Score: ${securityScore}/100 | Quality Gate Passed: ${isPassed}"

                        def statusColor = (isPassed == "true") ? "green" : "red"
                        def statusText = (isPassed == "true") ? "PASSED (SECURE)" : "FAILED (VULNERABLE)"

                        sh """
                        echo '<div class="header">' >> summary.html
                        echo '<h2>🛡️ Design File: ${file}</h2>' >> summary.html
                        echo '<p><strong>STRIDE Security Score:</strong> ${securityScore}/100</p>' >> summary.html
                        echo '<p><strong>Quality Gate Status:</strong> <span style="color: ${statusColor}; font-weight: bold;">${statusText}</span></p>' >> summary.html
                        echo '<p><strong>Total Threats:</strong> ${totalThreats}</p>' >> summary.html
                        echo '<a href="${reportPdf}" target="_blank" class="btn">📄 Download Full PDF Report</a>' >> summary.html
                        echo '</div>' >> summary.html
                        echo '<h3>Threat Details by Component:</h3>' >> summary.html
                        """

                        sh '''
                        jq -r '.threats[]? | "<div class=\\"threat-card \\(.severity)\\"><h4 style=\\"margin-top:0; color:#444;\\">🚨 \\(.componentName) - \\(.strideCategory) [\\(.severity)]</h4><p><strong>Finding:</strong> \\(.description)</p><p><strong>Technical Mitigation:</strong> <em>\\(.technicalMitigation)</em></p></div>"' response.json >> summary.html
                        '''

                        if (isPassed != "true") {
                            pipelineFailed = true
                        }
                    }

                    sh "echo '</body></html>' >> summary.html"

                    if (pipelineFailed) {
                        error("🚨 Pipeline stopped! Quality Gate FAILED because vulnerabilities were found in the design.")
                    } else {
                        echo "🎉 All designs passed the Quality Gate"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "📢 Publishing Visual Report..."
            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '', 
                reportFiles: 'summary.html',
                reportName: 'Security Threat Summary',
                reportTitles: 'The result of STRIDE Analysis for Design Files'
            ])
            
            sh "rm -f raw_response.txt response.json"
        }
    }
}
