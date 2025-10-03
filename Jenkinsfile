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
        stage('Set Branch') {
            steps {
                script {
                    echo "Using branch from parameter: ${params.GIT_BRANCH}"
                }
            }
        }

        stage('Checkout SCM') {
            steps {
                script {
                    echo "Checking out branch: ${params.GIT_BRANCH}"
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.GIT_BRANCH}"]],
                        userRemoteConfigs: [[url: 'git@github.com:infa-sasatapathy/terraform-vpc.git', credentialsId: 'jenkins']]
                    ])
                }
            }
        }

        stage('Init') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                    string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                ]) {
                    sh '''
                        terraform init -input=false
                        echo "Terraform initialisation completed"
                    '''
                }
            }
        }

        stage('Plan') {
            when { expression { params.TERRAFORM_ACTION == 'plan' || params.TERRAFORM_ACTION == 'apply' || params.TERRAFORM_ACTION == 'destroy' } }
            steps {
                script {
                    def planFile = "terraform-${params.ENVIRONMENT}-$(date +%s).tfplan"
                    echo "Creating Terraform plan for action: ${params.TERRAFORM_ACTION} in ${params.ENVIRONMENT} branch"

                    if (params.TERRAFORM_ACTION == 'destroy') {
                        sh """
                            terraform plan -destroy -out=${planFile} -input=false
                        """
                    } else {
                        sh """
                            terraform plan -out=${planFile} -input=false
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
                    input message: "Do you want to ${params.TERRAFORM_ACTION} the Terraform changes for branch ${params.GIT_BRANCH}?",
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
                    script {
                        echo "Applying Terraform ${params.TERRAFORM_ACTION} for ${params.ENVIRONMENT}"
                        sh """
                            terraform apply -auto-approve ${env.TF_PLAN_FILE}
                        """
                    }
                }
            }
        }

        stage('Completed') {
            steps {
                script {
                    echo "Terraform pipeline for ${params.ENVIRONMENT} completed with action: ${params.TERRAFORM_ACTION}"
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline executed successfully!"
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
