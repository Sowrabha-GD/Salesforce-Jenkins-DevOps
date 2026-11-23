
pipeline {
    agent any

    environment {
        SF_ORG = 'jenkins-sandbox'
       JIRA_SITE = 'sowrabhagd.atlassian.net'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('User Story Traceability') {
            steps {
                script {

                    def commitSubject = bat(
                        returnStdout: true,
                        script: 'git log -1 --format=%%s'
                    ).trim()

                    def commitId = bat(
                        returnStdout: true,
                        script: 'git rev-parse --short HEAD'
                    ).trim()

                    def branchName = bat(
                        returnStdout: true,
                        script: 'git branch --show-current'
                    ).trim()

                    // Existing User Story and GitHub Issue
                    def storyMatch = (commitSubject =~ /(US-\d+)/)
                    def issueMatch = (commitSubject =~ /#(\d+)/)

                    // Jira issue key, for example SDD-1
                    def jiraMatch = (commitSubject =~ /([A-Z][A-Z0-9]+-\d+)/)

                    def storyId = storyMatch.find() ?
                        storyMatch.group(1) : 'Not provided'

                    def issueId = issueMatch.find() ?
                        '#' + issueMatch.group(1) : 'Not provided'

                    def jiraIssueKey = jiraMatch.find() ?
                        jiraMatch.group(1) : 'Not provided'

                    currentBuild.displayName =
                        "#${env.BUILD_NUMBER} - ${storyId}"

                    echo "========== USER STORY DETAILS =========="
                    echo "User Story ID: ${storyId}"
                    echo "GitHub Issue: ${issueId}"
                    echo "Jira Issue Key: ${jiraIssueKey}"
                    echo "Git Branch: ${branchName}"
                    echo "Commit Message: ${commitSubject}"
                    echo "Commit ID: ${commitId}"
                    echo "Jenkins Build: ${env.BUILD_NUMBER}"
                    echo "Build URL: ${env.BUILD_URL}"
                    echo "========================================"

                }
            }
        }

        stage('Check Tools') {
            steps {
                bat 'git --version'
                bat 'sf --version'
            }
        }

        // Send build information to Jira
        stage('Send Build Information to Jira') {
            steps {
                echo 'Sending Jenkins build information to Jira...'
            }

            post {
                always {
                    jiraSendBuildInfo(
                        site: "${JIRA_SITE}"
                    )
                }
            }
        }

        stage('Authenticate Salesforce') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'salesforce-sandbox-auth-url',
                        variable: 'SF_AUTH_URL'
                    )
                ]) {
                    powershell '''
                        $authFile = Join-Path $env:WORKSPACE "sf-auth-url.txt"

                        try {
                            [System.IO.File]::WriteAllText(
                                $authFile,
                                $env:SF_AUTH_URL
                            )

                            sf org login sfdx-url `
                                --sfdx-url-file $authFile `
                                --alias jenkins-sandbox `
                                --set-default

                            if ($LASTEXITCODE -ne 0) {
                                exit $LASTEXITCODE
                            }
                        }
                        finally {
                            if (Test-Path $authFile) {
                                Remove-Item $authFile -Force
                            }
                        }
                    '''
                }
            }
        }

        stage('Deploy to Sandbox') {
            steps {
                bat 'sf project deploy start --source-dir force-app --target-org %SF_ORG% --test-level RunLocalTests --wait 30'
            }

            post {
                always {
                    jiraSendDeploymentInfo(
                        site: "${JIRA_SITE}",
                        environmentId: 'salesforce-dev',
                        environmentName: 'Salesforce DEV Sandbox',
                        environmentType: 'development'
                    )
                }
            }
        }
    }

    post {
        success {
            echo 'Salesforce deployment successful!'
        }

        failure {
            echo 'Pipeline failed. Check the console output.'
        }
    }
}
