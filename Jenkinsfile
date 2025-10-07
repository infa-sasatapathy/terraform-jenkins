pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stg', 'prod'],
            description: 'Select the environment (uses corresponding .tfvars file)'
        )
        choice(
            name: 'TERRAFORM_ACTION',
            choices: ['plan', 'apply', 'destroy'],
            description: 'Choose Terraform action'
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
                    echo "📦 Checking out Terraform repository..."
                    dir('infra') {  // ✅ clone into subfolder
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: "*/master"]],
                            userRemoteConfigs: [[
                                url: 'git@github.com:infa-sasatapathy/terraform-vpc.git',
                                credentialsId: 'jenkins'
                            ]]
                        ])
                    }
                    echo "✅ Terraform code checked out successfully into ./infra/"
                }
            }
        }

        stage('Validate & Fmt') {
            steps {
                dir(env.TERRAFORM_DIR) {
                    echo "🔍 Running terraform fmt & validate checks"
                    sh '''
                        # Auto-format everything (fixes exit code 3 issue)
                        terraform fmt -recursive
                        terraform validate
                      '''
        }
    }
}


        stage('Init') {
            steps {
                dir('infra') {
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
                dir('infra') {
                    script {
                        def timestamp = System.currentTimeMillis()
                        def planFile = "terraform-${env.ENVIRONMENT}-${timestamp}.tfplan"
                        def tfvarsFile = "${env.ENVIRONMENT}.tfvars"

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
                expression { params['TERRAFORM ACTION'] == 'plan' || params['TERRAFORM ACTION'] == 'apply' }
            }
            steps {
                script {
                    echo "Running Terratest for Terraform code in branch: ${params['GIT BRANCH']}"

                    // Assumes you have Go and Terratest installed on the Jenkins agent
                    // and your tests are inside the "test/" directory
                    sh '''
                        cd test
                        go mod tidy
                        go test -v -timeout 30m
                    '''
                }
            }
            post {
                always {
                    junit 'test/**/TEST-*.xml'  // if using gotestsum for JUnit XML output
                }
                failure {
                    error("Terratest failed! Fix tests before proceeding.")
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
                dir('infra') {
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
