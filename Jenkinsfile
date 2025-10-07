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

                    echo "📦 Checking out Terraform repository..."

                    // Try main first, then fallback to master
                    try {
                        dir(terraformDir) {
                            checkout([
                                $class: 'GitSCM',
                                branches: [[name: "*/main"]],
                                userRemoteConfigs: [[
                                    url: terraformRepo,
                                    credentialsId: 'jenkins'
                                ]]
                            ])
                        }
                    } catch (Exception e) {
                        echo "⚠️ 'main' branch not found, retrying with 'master'..."
                        dir(terraformDir) {
                            checkout([
                                $class: 'GitSCM',
                                branches: [[name: "*/master"]],
                                userRemoteConfigs: [[
                                    url: terraformRepo,
                                    credentialsId: 'jenkins'
                                ]]
                            ])
                        }
                    }

                    echo "✅ Terraform code checked out successfully into ./${terraformDir}/"
                    sh "ls -l ${terraformDir} || true"

                    // Save path for other stages
                    env.TERRAFORM_DIR = terraformDir
                }
            }
        }

        stage('Validate & Fmt') {
            steps {
                dir("${env.TERRAFORM_DIR}") {
                    echo "🔍 Running terraform fmt & validate checks"
                    sh '''
                        # Auto-format and validate
                        terraform fmt -recursive
                        terraform validate
                    '''
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

                        // Plan (or destroy plan)
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
                    sh '''
                        if [ -d "tests" ]; then
                            echo "Running Terratest Go tests..."
                            cd tests
                            go test -v ./... || echo "⚠️ Terratest failed (non-blocking)"
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
