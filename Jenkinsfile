pipeline {
    agent any

    parameters {
        choice(
            name: 'TERRAFORM_ACTION',
            choices: ['plan', 'apply', 'destroy'],
            description: 'Choose Terraform action: plan, apply or destroy'
        )
        string(
            name: 'AWS_DEFAULT_REGION',
            defaultValue: 'us-east-1',
            description: 'AWS Region for Terraform deployment'
        )
    }

    environment {
        AWS_DEFAULT_REGION = "${params.AWS_DEFAULT_REGION}"
        ENVIRONMENT        = "${params.GIT_BRANCH}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'jenkins', url: 'git@github.com:infa-sasatapathy/terraform-jenkins.git']])
                echo 'Code checked out successfully'
        }

        stage('Init') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    echo "Initializing Terraform..."
                    sh '''
                        terraform init -reconfigure
                        echo "Terraform initialization completed"
                    '''
                }
            }
        }

        stage('Plan') {
            when { expression { params.TERRAFORM_ACTION == 'plan' || params.TERRAFORM_ACTION == 'apply' || params.TERRAFORM_ACTION == 'destroy' } }
            steps {
                script {
                    def planFile = "terraform-${env.ENVIRONMENT}-$(date +%s).tfplan"
                    def tfvarsFile = "${env.ENVIRONMENT}.tfvars"

                    echo "Creating Terraform plan for ${env.ENVIRONMENT} using ${tfvarsFile}"

                    if (params.TERRAFORM_ACTION == 'destroy') {
                        sh """
                            terraform plan -destroy \
                                -var-file=${tfvarsFile} \
                                -out=${planFile} -input=false
                        """
                    } else {
                        sh """
                            terraform plan \
                                -var-file=${tfvarsFile} \
                                -out=${planFile} -input=false
                        """
                    }

                    archiveArtifacts artifacts: planFile, allowEmptyArchive: false
                    env.TF_PLAN_FILE = planFile
                }
            }
        }

        stage('Approvals') {
            when { expression { params.TERRAFORM_ACTION == 'apply' || params.TERRAFORM_ACTION == 'destroy' } }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    input message: "Do you want to ${params.TERRAFORM_ACTION} the Terraform changes for ${env.ENVIRONMENT}?",
                          ok: "Proceed with ${params.TERRAFORM_ACTION}"
                }
            }
        }

        stage('Apply') {
            when { expression { params.TERRAFORM_ACTION == 'apply' || params.TERRAFORM_ACTION == 'destroy' } }
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    echo "Applying Terraform ${params.TERRAFORM_ACTION} for ${env.ENVIRONMENT}"
                    sh """
                        terraform apply -var-file=${env.ENVIRONMENT}.tfvars -auto-approve ${env.TF_PLAN_FILE}
                    """
                }
            }
        }

        stage('Completed') {
            steps {
                echo "Terraform pipeline for ${env.ENVIRONMENT} completed with action: ${params.TERRAFORM_ACTION}"
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline executed successfully for ${env.ENVIRONMENT}!"
        }
        failure {
            echo "❌ Pipeline failed for ${env.ENVIRONMENT}!"
        }
    }
}
