
pipeline {
    agent any

    environment {
        SF_ORG = 'jenkins-sandbox'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Tools') {
            steps {
                bat 'git --version'
                bat 'sf --version'
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
