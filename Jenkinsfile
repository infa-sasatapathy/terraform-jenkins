pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stg', 'prod'],
            description: 'Select the environment (automatically picks matching .tfvars)'
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
                        echo "⚙️ Initializing Terraform in region ${env.AWS_DEFAULT_REGION}..."
                        sh '''
                            set -e
                            export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}
                            terraform init -reconfigure
                            echo "✅ Terraform initialization completed successfully in region ${AWS_DEFAULT_REGION}."
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
                        set -e
                        export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}
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

                        echo "🧩 Running Terraform ${params.TERRAFORM_ACTION} for ${env.ENVIRONMENT} in region ${env.AWS_DEFAULT_REGION}"

                        def planCmd = (params.TERRAFORM_ACTION == 'destroy') ?
                            "terraform plan -destroy -var-file=${tfvarsFile} -out=${planFile} -input=false" :
                            "terraform plan -var-file=${tfvarsFile} -out=${planFile} -input=false"

                        sh """
                            set -e
                            export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}
                            ${planCmd}
                        """

                        archiveArtifacts artifacts: planFile, allowEmptyArchive: false
                        env.TF_PLAN_FILE = planFile

                        // 🧹 Cleanup old plan files — keep last 3
                        echo "🧹 Cleaning up old Terraform plan files (keeping last 3)..."
                        sh '''
                            set -e
                            cd ${WORKSPACE}/${TERRAFORM_DIR}
                            ls -t terraform-*.tfplan | tail -n +4 | xargs -r rm -f || true
                            echo "✅ Old plan cleanup complete."
                        '''
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
                        export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}

                        if [ -d "tests" ]; then
                            echo "Running Terratest Go tests..."
                            cd tests

                            if [ ! -f "go.mod" ]; then
                                echo "⚙️ Initializing go.mod for Terratest..."
                                go mod init terratest
                                go get github.com/gruntwork-io/terratest/modules/terraform
                            fi

                            echo "▶️ Executing Terratest..."
                            go test -v ./... | tee terratest-output.txt
                            result=${PIPESTATUS[0]}

                            if [ $result -ne 0 ]; then
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
                        echo "🚀 Applying Terraform ${params.TERRAFORM_ACTION} for ${env.ENVIRONMENT} in ${env.AWS_DEFAULT_REGION}"
                        sh """
                            set -e
                            export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION}
                            terraform apply -var-file=${env.ENVIRONMENT}.tfvars -auto-approve ${env.TF_PLAN_FILE}
                        """
                    }
                }
            }
        }

        stage('Completed') {
            steps {
                echo "🎉 Terraform ${params.TERRAFORM_ACTION} completed successfully for ${env.ENVIRONMENT} in region ${env.AWS_DEFAULT_REGION}"
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
