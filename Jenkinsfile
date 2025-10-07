pipeline {
    agent any

    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'stg', 'prod'],
            description: 'Select the environment to deploy/destroy (chooses correct .tfvars file automatically)'
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
        ENVIRONMENT        = "${params.ENVIRONMENT}"
    }

    stages {

        stage('Checkout') {
            steps {
                script {
                    echo "📦 Checking out Terraform repo..."
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/main"]],
                        userRemoteConfigs: [[
                            url: 'git@github.com:infa-sasatapathy/terraform-vpc.git',
                            credentialsId: 'jenkins'
                        ]]
                    ])
                    echo "✅ Code checked out successfully"
                }
            }
        }

        stage('Validate & Fmt') {
            steps {
                dir('terraform-vpc') {
                    echo "🔍 Running terraform fmt & validate checks"
                    sh '''
                        terraform fmt -check
                        terraform validate
                    '''
                }
            }
        }

        stage('Init') {
            steps {
                dir('terraform-vpc') {
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
                dir('terraform-vpc') {
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
                    input message: "🟡 Do you want to ${params.TERRAFORM_ACTION} Terraform changes for ${env.ENVIRONMENT}?",
                          ok: "✅ Proceed with ${params.TERRAFORM_ACTION}"
                }
            }
        }

        stage('Apply') {
            when {
                expression { params.TERRAFORM_ACTION in ['apply', 'destroy'] }
            }
            steps {
                dir('terraform-vpc') {
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
                echo "🎉 Terraform pipeline for ${env.ENVIRONMENT} completed successfully with action: ${params.TERRAFORM_ACTION}"
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline executed successfully for ${env.ENVIRONMENT}!"
        }
        failure {
            echo "❌ Pipeline failed for ${env.ENVIRONMENT}!"
        }
        always {
            echo "📘 Jenkins Terraform pipeline run finished."
        }
    }
}
