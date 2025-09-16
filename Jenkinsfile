pipeline {
    agent slave

    environment {
        GH_TOKEN = credentials('github-token') 
    }

    stages {
        stage('Checkout') {
            steps {
                git(
                    branch: 'main',
                    url: 'https://github.com/itspavara/React-Movie-Site.git',
                    credentialsId: 'itspavara'
                )
            }
        }
        stage('Install GitHub CLI') {
            steps {
                sh '''
                if ! command -v gh &> /dev/null
                then
                    echo "Installing GitHub CLI..."
                    curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | \
                      sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
                    sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
                    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] \
                    https://cli.github.com/packages stable main" | \
                    sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
                    sudo apt update
                    sudo apt install gh -y
                else
                    echo "GitHub CLI already installed"
                fi
                '''
            }
}

        stage('Dependency Audit') {
            steps {
                nodejs('node-24.7') {
                    script {
                        if (fileExists('package-lock.json')) {
                            echo "Detected npm project → running npm audit"
                            sh 'npm install'
                            sh 'npm audit --json > audit-report.json || true'
                        } else if (fileExists('yarn.lock')) {
                            echo "Detected yarn project → running yarn audit"
                            sh 'yarn install --ignore-engines'
                            sh 'yarn audit --json > audit-report.json || true'
                        } else {
                            error "No package-lock.json or yarn.lock found — cannot determine package manager"
                        }
                    }
                }
            }
        }

        stage('Generate Markdown Report') {
            steps {
                script {
                    def finalVulns = []
                    def rawContent = readFile('audit-report.json')

                    if (fileExists('package-lock.json')) {
                        // NPM audit report
                        def npmReport = readJSON text: rawContent
                        if (npmReport.vulnerabilities) {
                            npmReport.vulnerabilities.each { name, vuln ->
                                if (vuln.severity in ['critical', 'high', 'moderate']) {
                                    finalVulns << [
                                        package : name,
                                        severity: vuln.severity,
                                        via     : vuln.via
                                    ]
                                }
                            }
                        }
                    } else if (fileExists('yarn.lock')) {
                        // Yarn audit report (line-delimited JSON)
                        rawContent.split("\n").each { line ->
                            if (line?.trim()) {
                                def obj = readJSON text: line
                                if (obj.type == "auditAdvisory") {
                                    def sev = obj.data.advisory.severity
                                    if (sev in ['critical', 'high', 'moderate']) {
                                        finalVulns << [
                                            package : obj.data.advisory.module_name,
                                            severity: sev,
                                            title   : obj.data.advisory.title
                                        ]
                                    }
                                }
                            }
                        }
                    }

                    if (finalVulns.size() > 0) {
                        def summary = "Found ${finalVulns.size()} critical/high/moderate vulnerabilities"
                        def details = finalVulns.collect { v ->
                            "- **${v.package}** (${v.severity}): ${v.title ?: v.via}"
                        }.join("\n")

                        writeFile file: 'summary.md', text: """
### Security Audit Report (Build #${BUILD_NUMBER})

${summary}

${details}

📎 Full JSON audit report is available in Jenkins artifacts.
"""
                    } else {
                        writeFile file: 'summary.md', text: """
### Security Audit Report (Build #${BUILD_NUMBER})

✅ No critical, high, or moderate vulnerabilities found.
"""
                    }
                }
            }
        }

        stage('Archive Report') {
            steps {
                archiveArtifacts artifacts: 'summary.md', fingerprint: true
                sh 'cat summary.md'
            }
        }

        stage('Create GitHub Issue') {
            steps {
                script {
                    def summaryContent = readFile('summary.md')
                    if (!summaryContent.contains("No critical, high, or moderate vulnerabilities")) {
                        sh '''
                          export GH_TOKEN=$GH_TOKEN
                          gh issue create \
                            --repo itspavara/React-Movie-Site \
                            --title "Security Audit Report - Build #${BUILD_NUMBER}" \
                            --body-file summary.md \
                            --label security \
                            --assignee itspavara
                        '''
                    } else {
                        echo "No issue created since no vulnerabilities were found."
                    }
                }
            }
        }
    }
}
