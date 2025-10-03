pipeline {
    agent any
    
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'stg', 'prod'])
        
        choice(
            name: 'TERRAFORM_ACTION',
            choices: ['apply', 'destroy'],
            description: 'Choose Terraform action: apply to create/update resources or destroy to remove all resources'
        )
    }
    
    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        TF_LOG = 'INFO'
        TF_LOG_PATH = 'terraform.log'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo 'Code checked out successfully'
            }
        }
        
        stage('Init') {
            steps {
                script {
                    // Set AWS credentials from Jenkins secret variables
                    withCredentials([string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                                   string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                        
                        env.AWS_ACCESS_KEY_ID = AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = AWS_SECRET_ACCESS_KEY
                        
                        echo 'Initializing Terraform...'
                        sh '''
                            terraform init -reconfigure -backend-config="env/${ENVIRONMENT}/backend.cfg"
                            echo "Terraform initialization completed"
                        '''
                    }
                }
            }
        }
        
        stage('Plan') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                                   string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                        
                        env.AWS_ACCESS_KEY_ID = AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = AWS_SECRET_ACCESS_KEY
                        
                        echo "Creating Terraform plan for action: ${params.TERRAFORM_ACTION}"
                        
                        if (params.TERRAFORM_ACTION == 'destroy') {
                            sh """
                                terraform plan -destroy -out=tfplan -var-file="env/${ENVIRONMENT}/terraform.tfvars"
                                echo "Terraform destroy plan created successfully"
                            """
                        } else {
                            sh """
                                terraform plan -out=tfplan -var-file="env/${ENVIRONMENT}/terraform.tfvars"
                                echo "Terraform apply plan created successfully"
                            """
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        // Archive the plan file and logs
                        archiveArtifacts artifacts: 'tfplan,terraform.log', allowEmptyArchive: true
                    }
                }
            }
        }
        
        stage('Ask Approvals') {
            steps {
                script {
                    def actionText = params.TERRAFORM_ACTION == 'destroy' ? 'destroy' : 'apply'
                    def buttonText = params.TERRAFORM_ACTION == 'destroy' ? 'Destroy Resources' : 'Apply Changes'
                    
                    echo "Waiting for manual approval to ${actionText} changes..."
                    input message: "Do you want to ${actionText} the Terraform changes?", ok: buttonText
                    echo "Approval received, proceeding to ${actionText} stage"
                }
            }
        }
        
        stage('Apply') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                                   string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                        
                        env.AWS_ACCESS_KEY_ID = AWS_ACCESS_KEY_ID
                        env.AWS_SECRET_ACCESS_KEY = AWS_SECRET_ACCESS_KEY
                        
                        def actionText = params.TERRAFORM_ACTION == 'destroy' ? 'destroying' : 'applying'
                        def successText = params.TERRAFORM_ACTION == 'destroy' ? 'destroy' : 'apply'
                        
                        echo "${actionText.capitalize()} Terraform changes..."
                        sh """
                            terraform apply -auto-approve tfplan
                            echo "Terraform ${params.TERRAFORM_ACTION} completed successfully"
                        """
                    }
                }
            }
            post {
                success {
                    script {
                        def actionText = params.TERRAFORM_ACTION == 'destroy' ? 'destruction' : 'deployment'
                        echo "Terraform ${actionText} completed successfully!"
                    }
                }
                failure {
                    script {
                        def actionText = params.TERRAFORM_ACTION == 'destroy' ? 'destruction' : 'deployment'
                        echo "Terraform ${actionText} failed!"
                    }
                }
                always {
                    script {
                        // Clean up the plan file
                        sh 'rm -f tfplan'
                        
                        // Archive final logs
                        archiveArtifacts artifacts: 'terraform.log', allowEmptyArchive: true
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline execution completed'
        }
        success {
            echo 'Pipeline executed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
