pipeline {
    agent any

    options {
        skipDefaultCheckout(true)  // avoid the hidden "Declarative: Checkout SCM"
    }

    parameters {
        choice(
            name: 'GIT BRANCH',
            choices: ['dev', 'stg', 'prod'],
            description: 'Select the Git branch (environment) to deploy/destroy'
        )
        choice(
            name: 'TERRAFORM ACTION',
            choices: ['plan', 'apply', 'destroy'],
            description: 'Choose Terraform action: plan, apply to create/update resources, or destroy to remove all resources'
        )
        string(
            name: 'AWS_DEFAULT_REGION',
            defaultValue: 'ap-south-1',
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
        ENVIRONMENT        = ""  // will be set in Set Default Branch stage
    }

    stages {
        stage('Set Default Branch') {
            steps {
                script {
                    def jobName = env.JOB_NAME.toLowerCase()
                    def defaultBranch = "dev" // fallback branch

                    if (jobName.contains("prod")) {
                        defaultBranch = "prod"
                    } else if (jobName.contains("stg")) {
                        defaultBranch = "stg"
                    } else if (jobName.contains("dev")) {
                        defaultBranch = "dev"
                    }

                    if (params['GIT BRANCH']) {
                        env.ENVIRONMENT = params['GIT BRANCH']
                        echo "Using user-selected GIT BRANCH: ${env.ENVIRONMENT}"
                    } else {
                        env.ENVIRONMENT = defaultBranch
                        echo "No GIT BRANCH provided, defaulting to: ${env.ENVIRONMENT} (based on job name)"
                    }

                    echo "DEBUG → params['GIT BRANCH'] = ${params['GIT BRANCH']}, env.ENVIRONMENT = ${env.ENVIRONMENT}"
                }
            }
        }

        stage('Checkout SCM') {
            steps {
                script {
                    echo "Checking out branch: ${env.ENVIRONMENT}"
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${env.ENVIRONMENT}"]],
                        userRemoteConfigs: [[
                            url: 'git@github.com:infa-sasatapathy/terraform-vpc.git',
                            credentialsId: 'jenkins'
                        ]]
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
                    '''
                }
            }
        }

        stage('Terratest') {
            when {
                expression { params['TERRAFORM ACTION'] == 'plan' || params['TERRAFORM ACTION'] == 'apply' }
            }
            steps {
                sh '''
                    cd test
                    go mod tidy
                    go test -v -timeout 30m
                '''
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
                    def tfplanFile = "terraform-${env.ENVIRONMENT}-${System.currentTimeMillis()}.tfplan"
                    env.TFPLAN_FILE = tfplanFile

                    echo "Creating plan for ${params['TERRAFORM ACTION']} in ${env.ENVIRONMENT}"

                    if (params['TERRAFORM ACTION'] == 'destroy') {
                        sh """
                            terraform plan -destroy -var="environment=${env.ENVIRONMENT}" -out=${tfplanFile} -input=false
                        """
                    } else {
                        sh """
                            terraform plan -var="environment=${env.ENVIRONMENT}" -out=${tfplanFile} -input=false
                        """
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: '*.tfplan,terraform.log', allowEmptyArchive: true
                }
            }
        }

        stage('Approvals') {
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

                    timeout(time: 1, unit: 'HOURS') {
                        input message: "Approve ${actionText} on ${env.ENVIRONMENT}?", ok: buttonText
                    }

                    if (params['TERRAFORM ACTION'] == 'destroy' && env.ENVIRONMENT == 'prod') {
                        timeout(time: 1, unit: 'HOURS') {
                            input message: "⚠️ FINAL CONFIRMATION: Destroy PRODUCTION (branch: prod)?", ok: "Yes, Destroy PRODUCTION"
                        }
                    }
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
                sh """
                    terraform apply -auto-approve ${env.TFPLAN_FILE}
                """
            }
        }

        stage('Completed') {
            steps {
                script {
                    if (currentBuild.currentResult == 'SUCCESS') {
                        slackSend(
                            channel: '#alerts',
                            color: 'good',
                            message: "✅ SUCCESS: Terraform ${params['TERRAFORM ACTION']} completed for *${env.ENVIRONMENT}*."
                        )
                        emailext(
                            subject: "✅ SUCCESS: Terraform ${params['TERRAFORM ACTION']} for ${env.ENVIRONMENT}",
                            body: "Pipeline succeeded for ${env.ENVIRONMENT}."
                        )
                    } else {
                        slackSend(
                            channel: '#alerts',
                            color: 'danger',
                            message: "❌ FAILURE: Terraform ${params['TERRAFORM ACTION']} failed for *${env.ENVIRONMENT}*."
                        )
                        emailext(
                            subject: "❌ FAILURE: Terraform ${params['TERRAFORM ACTION']} for ${env.ENVIRONMENT}",
                            body: "Pipeline failed for ${env.ENVIRONMENT}. Please check Jenkins logs."
                        )
                    }
                }
            }
        }
    }
}
