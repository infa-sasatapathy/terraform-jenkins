pipeline {
    agent any

    options {
        // Prevent Jenkins from doing its own hidden "Declarative: Checkout SCM"
        skipDefaultCheckout(true)
    }

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
        ENVIRONMENT        = "${params['GIT BRANCH']}"
    }

    stages {
        stage('Checkout SCM') {
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

        stage('Terratest') {
            when {
                expression { params['TERRAFORM ACTION'] == 'plan' || params['TERRAFORM ACTION'] == 'apply' }
            }
            steps {
                script {
                    echo "Running Terratest for Terraform code in branch: ${params['GIT BRANCH']}"

                    // Assumes you have Go + Terratest installed
                    sh '''
                        cd test
                        go mod tidy
                        go test -v -timeout 30m
                    '''
                }
            }
            post {
                failure {
                    error("Terratest failed! Fix tests before proceeding.")
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

                        def tfplanFile = "terraform-${params['GIT BRANCH']}-${System.currentTimeMillis()}.tfplan"
                        env.TFPLAN_FILE = tfplanFile

                        echo "Creating Terraform plan for action: ${params['TERRAFORM ACTION']} in branch: ${params['GIT BRANCH']}"

                        if (params['TERRAFORM ACTION'] == 'destroy') {
                            sh """
                                terraform plan -destroy -var="environment=${params['GIT BRANCH']}" -out=${tfplanFile} -input=false
                            """
                        } else {
                            sh """
                                terraform plan -var="environment=${params['GIT BRANCH']}" -out=${tfplanFile} -input=false
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

                    // First approval with timeout
                    timeout(time: 1, unit: 'HOURS') {
                        input message: "Do you want to ${actionText} the Terraform changes in branch: ${params['GIT BRANCH']}?", ok: buttonText
                    }

                    // Extra safeguard for production destroy
                    if (params['TERRAFORM ACTION'] == 'destroy' && params['GIT BRANCH'] == 'prd') {
                        echo "⚠️ Extra approval required for destroying resources in PRODUCTION!"
                        timeout(time: 1, unit: 'HOURS') {
                            input message: "FINAL CONFIRMATION: Do you REALLY want to DESTROY resources in PRODUCTION (branch: prd)?", ok: "Yes, Destroy PRODUCTION"
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
