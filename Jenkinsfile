pipeline {
    agent any

    environment {
        BACKEND_BASE_URL = 'http://172.235.251.236:8080'
        BACKEND_API_URL  = "${BACKEND_BASE_URL}/api/v1/analyze"
        // API_TOKEN = credentials('backend-api-token')
    }

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

                    def files = sh(
                        script: "git diff-tree --no-commit-id --name-only -r HEAD | grep -E '\\.(xml|puml|uml|jpg|jpeg|png)\$' || true",
                        returnStdout: true
                    ).trim()

                    // Fallback for shallow clones / first builds where the diff is empty.
                    if (files == "") {
                        echo "Git diff did not detect any changes. Scanning all design files..."
                        files = sh(
                            script: "find . -type f \\( -name '*.xml' -o -name '*.puml' -o -name '*.uml' -o -name '*.jpg' -o -name '*.png' \\) | sed 's|^./||' || true",
                            returnStdout: true
                        ).trim()
                    }

                    env.CHANGED_FILES = files

                    if (env.CHANGED_FILES == "") {
                        echo "No design files found. Skipping analysis."
                    } else {
                        echo "Design files to analyze:\n${env.CHANGED_FILES}"
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

                    sh "mkdir -p reports"

                    writeFile file: 'summary.html', text: '''<!DOCTYPE html>
<html><head><title>Design Security Report</title>
<style>
body{font-family:"Segoe UI",Tahoma,sans-serif;padding:20px;color:#333;line-height:1.6;}
.header{background:#f8f9fa;padding:20px;border-radius:8px;margin-bottom:20px;}
.threat-card{border:1px solid #ddd;padding:15px;margin-bottom:15px;border-radius:8px;}
.CRITICAL{border-left:6px solid #dc3545;background:#fff5f5;}
.HIGH{border-left:6px solid #fd7e14;background:#fff9f4;}
.MEDIUM{border-left:6px solid #ffc107;background:#fffdf5;}
.LOW{border-left:6px solid #17a2b8;}
.btn{display:inline-block;padding:10px 15px;background:#007bff;color:white;text-decoration:none;border-radius:5px;font-weight:bold;margin-top:10px;}
</style></head><body>
'''

                    for (int i = 0; i < fileArray.size(); i++) {
                        def file = fileArray[i].trim()
                        if (file == "") continue

                        echo "------------------------------------------------"
                        echo "Sending file to backend: ${file}"

                        // Body goes to response.json; HTTP code comes back via stdout.
                        // Cleaner than mixing body + status in one file and parsing them apart.
                        def httpStatus = sh(
                            script: """
                                curl -s -o response.json -w '%{http_code}' \
                                     --max-time 600 \
                                     -X POST '${BACKEND_API_URL}' \
                                     -F 'design_file=@${file}'
                            """,
                            returnStdout: true
                        ).trim()

                        if (httpStatus != "200") {
                            echo "ERROR: Backend returned HTTP ${httpStatus}"
                            sh "cat response.json || true"
                            pipelineFailed = true
                            continue
                        }

                        // Defensive jq: tolerate missing fields so partial responses don't crash the build.
                        def isPassed      = sh(script: "jq -r '.passedQualityGate // false' response.json", returnStdout: true).trim()
                        def reportPdfPath = sh(script: "jq -r '.downloadLinks.pdf // empty'  response.json", returnStdout: true).trim()
                        def securityScore = sh(script: "jq -r '.score // 0'                  response.json", returnStdout: true).trim()
                        def totalThreats  = sh(script: "jq    '.threats | length'            response.json", returnStdout: true).trim()

                        def passed = (isPassed == "true")
                        def statusColor = passed ? "green" : "red"
                        def statusText  = passed ? "PASSED (SECURE)" : "FAILED (VULNERABLE)"

                        echo "Score: ${securityScore}/100 | Quality Gate: ${statusText} | Threats: ${totalThreats}"

                        // Quality gate failed → download the PDF report and stash it as a Jenkins artifact.
                        def archivedPdf = ""
                        if (!passed && reportPdfPath) {
                            def pdfUrl   = "${BACKEND_BASE_URL}${reportPdfPath}"
                            def safeName = file.replaceAll('[^A-Za-z0-9._-]', '_')
                            def pdfFile  = "reports/${safeName}.pdf"

                            echo "Quality gate failed — downloading PDF report from ${pdfUrl}"
                            def pdfStatus = sh(
                                script: "curl -s -o '${pdfFile}' -w '%{http_code}' --max-time 300 '${pdfUrl}'",
                                returnStdout: true
                            ).trim()

                            if (pdfStatus == "200") {
                                echo "Saved PDF report to ${pdfFile}"
                                archivedPdf = pdfFile
                            } else {
                                echo "WARNING: PDF download failed (HTTP ${pdfStatus}) — ${pdfUrl}"
                                sh "rm -f '${pdfFile}'"
                            }
                        }

                        def reportLink = ""
                        if (archivedPdf) {
                            reportLink = "<a href=\"${env.BUILD_URL}artifact/${archivedPdf}\" target=\"_blank\" class=\"btn\">Download PDF Report</a>"
                        } else if (reportPdfPath) {
                            reportLink = "<a href=\"${BACKEND_BASE_URL}${reportPdfPath}\" target=\"_blank\" class=\"btn\">Download PDF Report</a>"
                        }

                        def header = """<div class="header">
  <h2>Design File: ${file}</h2>
  <p><strong>STRIDE Security Score:</strong> ${securityScore}/100</p>
  <p><strong>Quality Gate Status:</strong> <span style="color:${statusColor};font-weight:bold;">${statusText}</span></p>
  <p><strong>Total Threats:</strong> ${totalThreats}</p>
  ${reportLink}
</div>
<h3>Threat Details by Component:</h3>
"""
                        writeFile file: '_header.html', text: header
                        sh 'cat _header.html >> summary.html && rm -f _header.html'

                        sh '''
                        jq -r '.threats[]? | "<div class=\\"threat-card \\(.severity)\\"><h4 style=\\"margin-top:0;color:#444;\\">\\(.componentName) - \\(.strideCategory) [\\(.severity)]</h4><p><strong>Finding:</strong> \\(.description)</p><p><strong>Technical Mitigation:</strong> <em>\\(.technicalMitigation)</em></p></div>"' response.json >> summary.html
                        '''

                        if (!passed) {
                            pipelineFailed = true
                        }
                    }

                    sh "echo '</body></html>' >> summary.html"

                    if (pipelineFailed) {
                        error("Pipeline stopped: Quality Gate FAILED — vulnerabilities found in one or more design files.")
                    } else {
                        echo "All designs passed the Quality Gate."
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Publishing visual report..."
            publishHTML([
                allowMissing: true,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: '',
                reportFiles: 'summary.html',
                reportName: 'Security Threat Summary',
                reportTitles: 'STRIDE Analysis Result for Design Files'
            ])

            archiveArtifacts artifacts: 'reports/*.pdf', allowEmptyArchive: true, fingerprint: true

            sh "rm -f raw_response.txt response.json"
        }
    }
}
