pipeline {
    agent any
    environment {
        AWS_REGION = 'us-east-1'
        SONARQUBE_URL = "https://sonarcloud.io"
        TRUFFLEHOG_PATH = "/usr/local/bin/trufflehog3"
        JIRA_SITE = "https://jacquespayne@atlassian.net"
        JIRA_PROJECT = "JS" // Your Jira project key
    }

    stages {
        stage('Set AWS Credentials') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'AWS_SECRET_ACCESS_KEY' 
                ]]) {
                    sh '''
                    echo "AWS_ACCESS_KEY_ID: $AWS_ACCESS_KEY_ID"
                    aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/jdpayne68/autoscale.git'
            }
        }

        // Security Scans
        stage('Static Code Analysis (SAST)') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'sonarqube-api-key', variable: 'SONAR_TOKEN')]) {
                        def scanStatus = sh(script: '''
                            ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=jdpayne68_autoscale \
                            -Dsonar.organization=Kumo-Solutions \
                            -Dsonar.host.url=${SONARQUBE_URL} \
                            -Dsonar.token=''' + SONAR_TOKEN, returnStatus: true)

                        if (scanStatus != 0) {
                            createJiraTicket("Static Code Analysis Failed", "SonarQube scan detected issues in your code.")
                            error("SonarQube found security vulnerabilities!")
                        }
                    }
                }
            }
        }


        stage('Snyk Security Scan') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'snyk-api-key', variable: 'SNYK_TOKEN')]) {
                        sh "snyk auth ${SNYK_TOKEN}"
                        sh "snyk monitor || echo 'No supported files found, monitoring skipped.'"
                    }
                }
            }
        }


        stage('Initialize Terraform') {
            steps {
                sh '''
                terraform init
                '''
            }
        }


        stage('Plan Terraform') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'Jenkins-Server'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform plan -out=tfplan
                    '''
                }
            }
        }

        stage('Apply Terraform') {
            steps {
                input message: "Approve Terraform Apply?", ok: "Deploy"
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'Jenkins-Server'
                ]]) {
                    sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                    export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                    terraform apply -auto-approve tfplan
                    '''
                }
            }
        }


    }   

    post {
        success {
            echo 'Terraform deployment completed successfully!'
        }

        failure {
            echo 'Terraform deployment failed!'
        }
    }
}

// Function to Create a Jira Ticket
def createJiraTicket(String issueTitle, String issueDescription) {
    script {
        jiraNewIssue site: "${JIRA_SITE}",
                     projectKey: "${JIRA_PROJECT}",
                     issueType: "Bug",
                     summary: issueTitle,
                     description: issueDescription,
                     priority: "High"
    }
}
