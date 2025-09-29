pipeline {
    agent any

    parameters {
        choice(
            name: 'GIT BRANCH',
            choices: ['dev', 'stg', 'prd'],
            description: 'Select the Git branch (environment) to deploy/destroy'
        )
        choice(
            name: 'TERRAFORM ACTION',
            choices: ['plan', 'apply', 'destroy'],
            description: 'Choose Terraform action: plan, apply to create/update resources, or destroy to remove all resources'
        )
        string(
            name: 'AWS_DEFAULT_REGION',
            defaultValue: 'us-east-1',
            description: 'AWS Region for Terraform deployment'
        )
        choice(
            name: 'TF_LOG',
            choices: ['INFO', 'WARN', 'ERROR', 'DEBUG', 'TRACE'],
            description: 'Terraform log level'
        )
        string(
            name: 'TF_LOG_PATH',
            defaultValue: 'terraform.log',
            description: 'Path to store Terraform logs'
        )
    }

    environment {
        AWS_DEFAULT_REGION = "${params['AWS_DEFAULT_REGION']}"
        TF_LOG             = "${params['TF_LOG']}"
        TF_LOG_PATH        = "${params['TF_LOG_PATH']}"
        ENVIRONMENT        = "${params['GIT BRANCH']}"  // environment = selected branch
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out branch: ${params['GIT BRANCH']}"
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params['GIT BRANCH']}"]],
                        userRemoteConfigs: [[url: scm.userRemoteConfigs[0].url]]
                    ])
                }
            }
        }

        stage('Init') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        env.AWS_ACCESS_KEY_ID = AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = AWS_SECRET_ACCESS_KEY

                        echo "Initializing Terraform for branch/environment: ${params['GIT BRANCH']}"
                        sh '''
                            terraform init -input=false
                            echo "Terraform initialization completed"
                        '''
                    }
                }
            }
        }

        stage('Plan') {
            when {
                anyOf {
                    expression { params['TERRAFORM ACTION'] == 'plan' }
                    expression { params['TERRAFORM ACTION'] == 'apply' }
                    expression { params['TERRAFORM ACTION'] == 'destroy' }
                }
            }
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        env.AWS_ACCESS_KEY_ID = AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = AWS_SECRET_ACCESS_KEY

                        // unique tfplan filename
                        def tfplanFile = "terraform-${params['GIT BRANCH']}-${System.currentTimeMillis()}.tfplan"
                        env.TFPLAN_FILE = tfplanFile

                        echo "Creating Terraform plan for action: ${params['TERRAFORM ACTION']} in branch: ${params['GIT BRANCH']}"

                        if (params['TERRAFORM ACTION'] == 'destroy') {
                            sh """
                                terraform plan -destroy -var="environment=${params['GIT BRANCH']}" -out=${tfplanFile} -input=false
                                echo "Terraform destroy plan created successfully: ${tfplanFile}"
                            """
                        } else {
                            sh """
                                terraform plan -var="environment=${params['GIT BRANCH']}" -out=${tfplanFile} -input=false
                                echo "Terraform plan created successfully: ${tfplanFile}"
                            """
                        }
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: '*.tfplan,terraform.log', allowEmptyArchive: true
                }
            }
        }

        stage('Ask Approvals') {
            when {
                anyOf {
                    expression { params['TERRAFORM ACTION'] == 'apply' }
                    expression { params['TERRAFORM ACTION'] == 'destroy' }
                }
            }
            steps {
                script {
                    def actionText = params['TERRAFORM ACTION']
                    def buttonText = actionText == 'destroy' ? 'Destroy Resources' : 'Apply Changes'

                    echo "Waiting for manual approval to ${actionText} changes in branch: ${params['GIT BRANCH']}..."

                    // 🔔 Send notification if it's a destroy in prd
                    if (params['TERRAFORM ACTION'] == 'destroy' && params['GIT BRANCH'] == 'prd') {
                        echo "⚠️ ALERT: Destroy requested for PRODUCTION!"
                        
                        // Slack notification (requires Jenkins Slack plugin configured)
                        slackSend(
                            channel: '#alerts',
                            color: 'danger',
                            message: "⚠️ Jenkins Alert: Terraform DESTROY requested on *PRODUCTION* (branch: prd).\nJob: ${env.JOB_NAME} #${env.BUILD_NUMBER}\nTriggered by: ${currentBuild.getBuildCauses()[0].userId}"
                        )

                        // Email notification (requires Jenkins Mailer configured)
                        emailext(
                            subject: "⚠️ ALERT: Terraform DESTROY requested on PRODUCTION",
                            body: """A Jenkins job has requested Terraform DESTROY on PRODUCTION.

Job: ${env.JOB_NAME} #${env.BUILD_NUMBER}
Branch: ${params['GIT BRANCH']}
Action: ${params['TERRAFORM ACTION']}
Triggered by: ${currentBuild.getBuildCauses()[0].userId}

Please review the job in Jenkins before approving.
""",
                            to: "devops-team@example.com"
                        )
                    }

                    // First approval
                    input message: "Do you want to ${actionText} the Terraform changes in branch: ${params['GIT BRANCH']}?", ok: buttonText

                    // 🔒 Extra safeguard: require second approval for destroy in prd
                    if (params['TERRAFORM ACTION'] == 'destroy' && params['GIT BRANCH'] == 'prd') {
                        echo "⚠️ Extra approval required for destroying resources in PRODUCTION!"
                        input message: "FINAL CONFIRMATION: Do you REALLY want to DESTROY resources in PRODUCTION (branch: prd)?", ok: "Yes, Destroy PRODUCTION"
                    }

                    echo "Approval(s) received, proceeding to ${actionText} stage"
                }
            }
        }

        stage('Apply') {
            when {
                anyOf {
                    expression { params['TERRAFORM ACTION'] == 'apply' }
                    expression { params['TERRAFORM ACTION'] == 'destroy' }
                }
            }
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        env.AWS_ACCESS_KEY_ID = AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = AWS_SECRET_ACCESS_KEY

                        def actionText = params['TERRAFORM ACTION'] == 'destroy' ? 'destroying' : 'applying'

                        echo "${actionText.capitalize()} Terraform changes for branch: ${params['GIT BRANCH']} using plan file: ${env.TFPLAN_FILE}..."
                        sh """
                            terraform apply -auto-approve ${env.TFPLAN_FILE}
                            echo "Terraform ${params['TERRAFORM ACTION']} completed successfully"
                        """
                    }
                }
            }
            post {
                success {
                    echo "Terraform ${params['TERRAFORM ACTION']} for branch: ${params['GIT BRANCH']} completed successfully!"
                }
                failure {
                    echo "Terraform ${params['TERRAFORM ACTION']} for branch: ${params['GIT BRANCH']} failed!"
                }
                always {
                    sh 'rm -f *.tfplan'
                    archiveArtifacts artifacts: 'terraform.log', allowEmptyArchive: true
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline execution for branch: ${params['GIT BRANCH']} completed"
        }
        success {
            echo 'Pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
