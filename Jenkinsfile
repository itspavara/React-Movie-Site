pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
              git branch: 'main', 
              url: 'https://github.com/itspavara/React-Movie-Site.git',
              credentialsId: 'itspavara' // saved in Jenkins
            }
        }

        stage('Dependency Audit') {
            steps {
                sh 'npm install'
                sh 'npm audit --json > audit-report.json || true'
            }
        }

      stage('check report'){
            steps {
              sh 'cat audit-report.json'
            }
        
      }

        // stage('Create GitHub Issue') {
        //     steps {
        //         script {
        //             def report = readJSON file: 'audit-report.json'
        //             if (report.metadata.totalDependencies > 0 && report.vulnerabilities.total > 0) {
        //                 def summary = "Found ${report.vulnerabilities.total} vulnerabilities"
        //                 def details = report.advisories.collect { adv -> 
        //                     "- ${adv.module_name} (${adv.severity}): ${adv.title}"
        //                 }.join("\n")
        //                 sh """
        //                   gh issue create \
        //                     --repo your-org/your-node-app \
        //                     --title "Dependency Vulnerabilities Report" \
        //                     --body "${summary}\n\n${details}" \
        //                     --assignee dev_username
        //                 """
        //             }
        //         }
        //     }
        // }
    }
}  
