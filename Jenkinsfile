pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
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
            description: 'Choose Terraform action'
        )
        string(
            name: 'AWS_DEFAULT_REGION',
            defaultValue: 'ap-south-1',
            description: 'AWS Region'
        )
        choice(
            name: 'TF_LOG',
            choices: ['INFO', 'WARN', 'ERROR', 'DEBUG', 'TRACE'],
            description: 'Terraform log level'
        )
        string(
            name: 'TF_LOG_PATH',
            defaultValue: 'terraform.log',
            description: 'Terraform log path'
        )
    }

    environment {
        AWS_DEFAULT_REGION = "${params['AWS_DEFAULT_REGION']}"
        TF_LOG             = "${params['TF_LOG']}"
        TF_LOG_PATH        = "${params['TF_LOG_PATH']}"
    }

    stages {
        stage('Set Branch') {
            steps {
                script {
                    echo "Using branch from parameter: ${params['GIT BRANCH']}"
                }
            }
        }

        stage('Checkout SCM') {
            steps {
                script {
                    def branchName = params['GIT BRANCH'].toString()
                    echo "Checking out branch: ${branchName}"
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${branchName}"]],
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
                    sh 'terraform init -input=false'
                }
            }
        }

        stage('Plan') {
            steps {
                script {
                    def branchName = params['GIT BRANCH'].toString()
                    def action = params['TERRAFORM ACTION'].toString()
                    def tfplanFile = "terraform-${branchName}-${System.currentTimeMillis()}.tfplan"
                    env.TFPLAN_FILE = tfplanFile

                    if (action == "destroy") {
                        sh """
                          terraform plan -destroy -var="environment=${branchName}" -out=${tfplanFile} -input=false
                        """
                    } else {
                        sh """
                          terraform plan -var="environment=${branchName}" -out=${tfplanFile} -input=false
                        """
                    }
                }
            }
        }

        stage('Approvals') {
            when {
                anyOf {
                    expression { params['TERRAFORM ACTION'].toString() == 'apply' }
                    expression { params['TERRAFORM ACTION'].toString() == 'destroy' }
                }
            }
            steps {
                script {
                    def action = params['TERRAFORM ACTION'].toString()
                    timeout(time: 1, unit: 'HOURS') {
                        input message: "Approve ${action} on ${params['GIT BRANCH']}?", ok: "Proceed"
                    }
                }
            }
        }

        stage('Apply') {
            when {
                anyOf {
                    expression { params['TERRAFORM ACTION'].toString() == 'apply' }
                    expression { params['TERRAFORM ACTION'].toString() == 'destroy' }
                }
            }
            steps {
                sh "terraform apply -auto-approve ${env.TFPLAN_FILE}"
            }
        }

        stage('Completed') {
            steps {
                script {
                    if (currentBuild.currentResult == 'SUCCESS') {
                        echo "✅ SUCCESS: Terraform ${params['TERRAFORM ACTION']} completed for ${params['GIT BRANCH']}"
                    } else {
                        echo "❌ FAILURE: Terraform ${params['TERRAFORM ACTION']} failed for ${params['GIT BRANCH']}"
                    }
                }
            }
        }
    }
}
