pipeline {
    agent any

    parameters {
        choice(
            name: 'GIT_BRANCH',
            choices: ['dev', 'stg', 'prod'],
            description: 'Select the Git branch (environment) to deploy/destroy'
        )
        choice(
            name: 'TERRAFORM_ACTION',
            choices: ['apply', 'destroy'],
            description: 'Choose Terraform action: apply to create/update resources or destroy to remove all resources'
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
        AWS_DEFAULT_REGION = "${params.AWS_DEFAULT_REGION}"
        TF_LOG             = "${params.TF_LOG}"
        TF_LOG_PATH        = "${params.TF_LOG_PATH}"
        ENVIRONMENT        = "${params.GIT_BRANCH}"  // environment = selected branch
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out branch: ${params.GIT_BRANCH}"
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.GIT_BRANCH}"]],
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

                        echo "Initializing Terraform for branch/environment: ${params.GIT_BRANCH}"
                        sh '''
                            terraform init -input=false
                            echo "Terraform initialization completed"
                        '''
                    }
                }
            }
        }

        stage('Plan') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        env.AWS_ACCESS_KEY_ID = AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = AWS_SECRET_ACCESS_KEY

                        echo "Creating Terraform plan for action: ${params.TERRAFORM_ACTION} in branch: ${params.GIT_BRANCH}"

                        if (params.TERRAFORM_ACTION == 'destroy') {
                            sh """
                                terraform plan -destroy -var="environment=${params.GIT_BRANCH}" -out=tfplan -input=false
                                echo "Terraform destroy plan created successfully"
                            """
                        } else {
                            sh """
                                terraform plan -var="environment=${params.GIT_BRANCH}" -out=tfplan -input=false
                                echo "Terraform apply plan created successfully"
                            """
                        }
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'tfplan,terraform.log', allowEmptyArchive: true
                }
            }
        }

        stage('Ask Approvals') {
            steps {
                script {
                    def actionText = params.TERRAFORM_ACTION == 'destroy' ? 'destroy' : 'apply'
                    def buttonText = params.TERRAFORM_ACTION == 'destroy' ? 'Destroy Resources' : 'Apply Changes'

                    echo "Waiting for manual approval to ${actionText} changes in branch: ${params.GIT_BRANCH}..."
                    input message: "Do you want to ${actionText} the Terraform changes in branch: ${params.GIT_BRANCH}?", ok: buttonText
                    echo "Approval received, proceeding to ${actionText} stage"
                }
            }
        }

        stage('Apply') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        env.AWS_ACCESS_KEY_ID = AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = AWS_SECRET_ACCESS_KEY

                        def actionText = params.TERRAFORM_ACTION == 'destroy' ? 'destroying' : 'applying'

                        echo "${actionText.capitalize()} Terraform changes for branch: ${params.GIT_BRANCH}..."
                        sh """
                            terraform apply -auto-approve tfplan
                            echo "Terraform ${params.TERRAFORM_ACTION} completed successfully"
                        """
                    }
                }
            }
            post {
                success {
                    echo "Terraform ${params.TERRAFORM_ACTION} for branch: ${params.GIT_BRANCH} completed successfully!"
                }
                failure {
                    echo "Terraform ${params.TERRAFORM_ACTION} for branch: ${params.GIT_BRANCH} failed!"
                }
                always {
                    sh 'rm -f tfplan'
                    archiveArtifacts artifacts: 'terraform.log', allowEmptyArchive: true
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline execution for branch: ${params.GIT_BRANCH} completed"
        }
        success {
            echo 'Pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
