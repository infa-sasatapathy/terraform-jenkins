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
        stage('Checkout') {
            steps {
                script {
                    echo "Checking out branch: ${params.GIT_BRANCH}"
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${params.GIT_BRANCH}"]],
                        userRemoteConfigs: [[url: 'git@github.com:infa-sasatapathy/terraform-vpc.git', credentialsId: 'jenkins']]
                    ])
                    echo "Code checked out successfully for branch: ${params.GIT_BRANCH}"
                }
            }
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
            when { expression { params.TERRAFORM_ACTION in ['plan', 'apply', 'destroy'] } }
            steps {
                script {
                    // ✅ Generate timestamp safely in Groovy, not bash
                    def timestamp = System.currentTimeMillis()
                    def planFile = "terraform-${env.ENVIRONMENT}-${timestamp}.tfplan"
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
            when { expression { params.TERRAFORM_ACTION in ['apply', 'destroy'] } }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    input message: "Do you want to ${params.TERRAFORM_ACTION} the Terraform changes for ${env.ENVIRONMENT}?",
                          ok: "Proceed with ${params.TERRAFORM_ACTION}"
                }
            }
        }

        stage('Apply')
