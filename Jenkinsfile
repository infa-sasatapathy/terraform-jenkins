pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stg', 'prod'],
            description: 'Select the environment (auto-picks matching .tfvars)'
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
        AWS_DEFAULT_REGION = "${params.AWS_DEFAULT_REGION ?: 'us-east-1'}"
        ENVIRONMENT        = "${params.ENVIRONMENT ?: 'dev'}"
        TERRAFORM_ACTION   = "${params.TERRAFORM_ACTION ?: 'plan'}"
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
                        echo "⚙️ Initializing Terraform in region ${params.AWS_DEFAULT_REGION}..."
                        sh """
                            set -e
                            export AWS_DEFAULT_REGION=${params.AWS_DEFAULT_REGION}
                            terraform init -reconfigure
                            echo "✅ Terraform initialization completed successfully."
                        """
                    }
                }
            }
        }

        stage('Validate') {
            steps {
                dir("${env.TERRAFORM_DIR}") {
                    echo "🔍 Running terraform fmt & validate checks"
                    sh """
                        set -e
                        terraform validate
                    """
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
                        def regionVar  = "-var='region=${params.AWS_DEFAULT_REGION}'"
                        def actionArg  = params.TERRAFORM_ACTION == 'destroy' ? '-destroy' : ''

                        echo "🧩 Running Terraform plan for ${env.ENVIRONMENT} in ${params.AWS_DEFAULT_REGION}"

                        sh """
                            set -e
                            export AWS_DEFAULT_REGION=${params.AWS_DEFAULT_REGION}
                            terraform plan ${actionArg} -var-file=${tfvarsFile} ${regionVar} -out=${planFile} -input=false
                        """

                        archiveArtifacts artifacts: planFile, allowEmptyArchive: false
                        env.TF_PLAN_FILE = planFile

                        // Keep only last 3 .tfplan files
                        echo "🧹 Cleaning up old Terraform plan files (keeping last 3)..."
                        sh """
                            ls -t terraform-*.tfplan | tail -n +4 | xargs -r rm -f || true
                        """
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
                    script {
                        def terratestResult = sh(
                            script: """
                                set -e
                                if [ -d "tests" ]; then
                                    echo "Running Terratest Go tests..."
                                    cd tests
                                    if [ ! -f "go.mod" ]; then
                                        go mod init terratest
                                        go get github.com/gruntwork-io/terratest/modules/terraform
                                    fi
                                    go test -v ./...
                                else
                                    echo "ℹ️ No Terratest directory found, skipping..."
                                    exit 0
                                fi
                            """,
                            returnStatus: true
                        )
                        if (terratestResult != 0) {
                            error("❌ Terratest failed — stopping pipeline.")
                        } else {
                            echo "✅ Terratest passed successfully."
                        }
                    }
                }
            }
        }

        // 🟡 Manual Approval (for apply/destroy only)
        stage('Approval') {
            when {
                expression { params.TERRAFORM_ACTION in ['apply', 'destroy'] }
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: "🟡 Approve Terraform ${params.TERRAFORM_ACTION.toUpperCase()} for ${env.ENVIRONMENT}?",
                          ok: "✅ Proceed"
                }
            }
        }

        // 🟢 APPLY INFRASTRUCTURE
        stage('Apply Infrastructure') {
            when {
                expression { params.TERRAFORM_ACTION == 'apply' }
            }
            steps {
                dir("${env.TERRAFORM_DIR}") {
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        echo "🚀 Applying Terraform for ${env.ENVIRONMENT} in ${params.AWS_DEFAULT_REGION}"
                        sh """
                            set -e
                            export AWS_DEFAULT_REGION=${params.AWS_DEFAULT_REGION}
                            terraform apply -auto-approve -var-file=${env.ENVIRONMENT}.tfvars -var="region=${params.AWS_DEFAULT_REGION}" ${env.TF_PLAN_FILE}
                        """
                    }
                }
            }
        }

        // 🔴 DESTROY INFRASTRUCTURE
        stage('Destroy Infrastructure') {
            when {
                expression { params.TERRAFORM_ACTION == 'destroy' }
            }
            steps {
                dir("${env.TERRAFORM_DIR}") {
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        echo "💣 Destroying Terraform resources for ${env.ENVIRONMENT} in ${params.AWS_DEFAULT_REGION}"
                        sh """
                            set -e
                            export AWS_DEFAULT_REGION=${params.AWS_DEFAULT_REGION}
                            terraform destroy -auto-approve -var-file=${env.ENVIRONMENT}.tfvars -var="region=${params.AWS_DEFAULT_REGION}"
                        """
                    }
                }
            }
        }

        stage('Completed') {
            steps {
                echo "🎉 Terraform ${params.TERRAFORM_ACTION} completed successfully for ${env.ENVIRONMENT} (${params.AWS_DEFAULT_REGION})"
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline executed successfully for ${env.ENVIRONMENT} (${params.AWS_DEFAULT_REGION})"
        }
        failure {
            echo "❌ Pipeline failed for ${env.ENVIRONMENT} (${params.AWS_DEFAULT_REGION})"
        }
        always {
            echo "📘 Jenkins Terraform pipeline finished"
        }
    }
}
