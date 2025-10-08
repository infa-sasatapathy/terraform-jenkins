pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stg', 'prod'],
            description: 'Select the environment'
        )
        choice(
            name: 'TERRAFORM_ACTION',
            choices: ['plan', 'apply', 'destroy'],
            description: 'Choose Terraform action to perform'
        )
        string(
            name: 'AWS_DEFAULT_REGION',
            defaultValue: 'us-east-1',
            description: 'AWS region for Terraform deployment'
        )
    }

    environment {
        AWS_DEFAULT_REGION = "${params.AWS_DEFAULT_REGION}"
        ENVIRONMENT        = "${params.ENVIRONMENT}"
    }

    stages {

        stage('Checkout Terraform Repo') {
            steps {
                script {
                    def terraformDir = "infra"
                    def terraformRepo = "git@github.com:infa-sasatapathy/terraform-vpc.git"
                    def terraformBranch = "master"

                    echo "📦 Checking out Terraform repository from '${terraformBranch}' branch..."

                    dir(terraformDir) {
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: "*/${terraformBranch}"]],
                            userRemoteConfigs: [[
                                url: terraformRepo,
                                credentialsId: 'jenkins'
                            ]]
                        ])
                    }

                    echo "✅ Terraform code checked out successfully into ./${terraformDir}/"
                    sh "ls -l ${terraformDir} || true"

                    env.TERRAFORM_DIR = terraformDir
                }
            }
        }

        stage('Init') {
            steps {
                dir("${env.TERRAFORM_DIR}") {
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        echo "⚙️ Initializing Terraform..."
                        sh '''
                            terraform init -reconfigure
                            echo "Terraform initialization completed successfully."
                        '''
                    }
                }
            }
        }

        stage('Validate & Fmt') {
            steps {
                dir("${env.TERRAFORM_DIR}") {
                    echo "🔍 Running terraform fmt & validate checks"
                    sh '''
                        terraform fmt -recursive
                        terraform validate
                    '''
                }
            }
        }

        stage('Plan') {
            when {
                expression { params.TERRAFORM_ACTION in ['plan', 'apply', 'destroy'] }
            }
            steps {
                dir("${env.TERRAFORM_DIR}") {
                    script {
                        def tfvarsFile = "${env.ENVIRONMENT}.tfvars"
                        def timestamp  = System.currentTimeMillis()
                        def planFile   = "terraform-${env.ENVIRONMENT}-${timestamp}.tfplan"

                        echo "🧩 Running Terraform plan for ${env.ENVIRONMENT} using ${tfvarsFile}"

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
        }

stage('Terratest') {
    when {
    expression { params.TERRAFORM_ACTION == 'plan' }
    }
    steps {
    dir("${env.TERRAFORM_DIR}") {
        echo "🧪 Running Terratest for ${env.ENVIRONMENT}"
        sh '''#!/bin/bash
        set -e

        if [ -d "tests" ]; then
            echo "Running Terratest Go tests..."
            cd tests

            # Initialize module if missing
            if [ ! -f "go.mod" ]; then
            echo "⚙️ Initializing go.mod for Terratest..."
            go mod init terratest
            go get github.com/gruntwork-io/terratest/modules/terraform
            fi

            # Run Terratest (fail pipeline if tests fail)
            echo "▶️ Executing Terratest..."
            go test -v ./... | tee terratest-output.txt
            test_result=${PIPESTATUS[0]}

            if [ $test_result -ne 0 ]; then
            echo "❌ Terratest failed! Stopping pipeline."
            exit 1
            else
            echo "✅ Terratest passed successfully!"
            fi
        else
            echo "ℹ️ No Terratest directory found, skipping..."
        fi
        '''
    }
    }
}

        stage('Approvals') {
            when {
                expression { params.TERRAFORM_ACTION in ['apply', 'destroy'] }
            }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    input message: "🟡 Approve Terraform ${params.TERRAFORM_ACTION} for ${env.ENVIRONMENT}?",
                          ok: "✅ Proceed"
                }
            }
        }

        stage('Apply') {
            when {
                expression { params.TERRAFORM_ACTION in ['apply', 'destroy'] }
            }
            steps {
                dir("${env.TERRAFORM_DIR}") {
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        echo "🚀 Applying Terraform ${params.TERRAFORM_ACTION} for ${env.ENVIRONMENT}"
                        sh """
                            terraform apply -var-file=${env.ENVIRONMENT}.tfvars -auto-approve ${env.TF_PLAN_FILE}
                        """
                    }
                }
            }
        }

        stage('Completed') {
            steps {
                echo "🎉 Terraform ${params.TERRAFORM_ACTION} completed successfully for ${env.ENVIRONMENT}"
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline executed successfully for ${env.ENVIRONMENT}"
        }
        failure {
            echo "❌ Pipeline failed for ${env.ENVIRONMENT}"
        }
        always {
            echo "📘 Jenkins Terraform pipeline finished"
        }
    }
}
