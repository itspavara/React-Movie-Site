pipeline {
    agent any

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

        stage('Filter Vulnerabilities') {
            steps {
                script {
                    def finalVulns = []

                    // Load the audit report
                    def rawContent = readFile('audit-report.json')

                    if (fileExists('package-lock.json')) {
                        // NPM report
                        def npmReport = readJSON text: rawContent
                        if (npmReport.vulnerabilities) {
                            npmReport.vulnerabilities.each { name, vuln ->
                                if (vuln.severity in ['high', 'moderate']) {
                                    finalVulns << [
                                        package: name,
                                        severity: vuln.severity,
                                        via: vuln.via
                                    ]
                                }
                            }
                        }
                    } else if (fileExists('yarn.lock')) {
                        // Yarn report (line-delimited JSON)
                        rawContent.split("\n").each { line ->
                            if (line?.trim()) {
                                def obj = readJSON text: line
                                if (obj.type == "auditAdvisory") {
                                    def sev = obj.data.advisory.severity
                                    if (sev in ['high', 'moderate']) {
                                        finalVulns << [
                                            package: obj.data.advisory.module_name,
                                            severity: sev,
                                            title: obj.data.advisory.title
                                        ]
                                    }
                                }
                            }
                        }
                    }

                    // filtered report
                    def filteredReport = [ vulnerabilities: finalVulns ]
                    writeJSON file: 'filtered-audit-report.json', json: filteredReport, pretty: 4
                }
            }
        }

        stage('Archive Report') {
            steps {
                archiveArtifacts artifacts: 'filtered-audit-report.json', fingerprint: true
                sh 'cat filtered-audit-report.json'
            }
        }
    }
}
