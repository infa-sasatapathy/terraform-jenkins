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
        string(
            name: 'TERRAFORM_REPO',
            defaultValue: 'git@github.com:infa-sasatapathy/terraform-vpc.git',
            description: 'Git repository URL for Terraform code'
        )
        string(
            name: 'TERRAFORM_BRANCH',
            defaultValue: 'main',
            description: 'Git branch to checkout (e.g., main or master)'
        )
        string(
            name: 'TERRAFORM_DIR',
            defaultValue: 'infra',
            description: 'Local directory name for Terraform code'
        )
    }

    environment {
        AWS_DEFAULT_REGION = "${params.AWS_DEFAULT_REGION ?: 'us-east-1'}"
        ENVIRONMENT        = "${params.ENVIRONMENT ?: 'dev'}"
        TERRAFORM_ACTION   = "${params.TERRAFORM_ACTION ?: 'plan'}"
        TERRAFORM_REPO     = "${params.TERRAFORM_REPO}"
        TERRAFORM_BRANCH   = "${params.TERRAFORM_BRANCH}"
        TERRAFORM_DIR      = "${params.TERRAFORM_DIR}"
    }

    stages {

        stage('Checkout Terraform Repo') {
            steps {
                script {
                    echo "📦 Checking out Terraform repository: ${params.TERRAFORM_REPO} (branch: ${params.TERRAFORM_BRANCH})"

                    dir("${params.TERRAFORM_DIR}") {
                        checkout([
                            $class: 'GitSCM',
                            branches: [[name: "*/${params.TERRAFORM_BRANCH}"]],
                            userRemoteConfigs: [[
                                url: params.TERRAFORM_REPO,
                                credentialsId: 'jenkins'
                            ]]
                        ])
                    }

                    echo "✅ Checked out ${params.TERRAFORM_BRANCH} branch into ./${params.TERRAFORM_DIR}/"
                    sh "ls -l ${params.TERRAFORM_DIR} || true"
                }
            }
        }

        stage('Init') {
            steps {
                dir("${params.TERRAFORM_DIR}") {
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

        stage('Validate & Fmt') {
            steps {
                dir("${params.TERRAFORM_DIR}") {
                    echo "🔍 Running terraform fmt & validate checks"
                    sh """
                        set -e
                        terraform fmt -recursive
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
                dir("${params.TERRAFORM_DIR}") {
                    script {
                        def tfvarsFile = "${params.ENVIRONMENT}.tfvars"
                        def timestamp  = System.currentTimeMillis()
                        def planFile   = "terraform-${params.ENVIRONMENT}-${timestamp}.tfplan"
                        def regionVar  = "-var='region=${params.AWS_DEFAULT_REGION}'"
                        def actionArg  = params.TERRAFORM_ACTION == 'destroy' ? '-destroy' : ''

                        echo "🧩 Running Terraform plan for ${params.ENVIRONMENT} in ${params.AWS_DEFAULT_REGION}"

                        sh """
                            set -e
                            export AWS_DEFAULT_REGION=${params.AWS_DEFAULT_REGION}
                            terraform plan ${actionArg} -var-file=${tfvarsFile} ${regionVar} -out=${planFile} -input=false
                        """

                        archiveArtifacts artifacts: planFile, allowEmptyArchive: false
                        env.TF_PLAN_FILE = planFile

                        // Keep only last 3 plans
                        echo "🧹 Cleaning up old Terraform plan files..."
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
                dir("${params.TERRAFORM_DIR}") {
                    echo "🧪 Running Terratest for ${params.ENVIRONMENT}"
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

        stage('Approval') {
            when {
                expression { params.TERRAFORM_ACTION in ['apply', 'destroy'] }
            }
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: "🟡 Approve Terraform ${params.TERRAFORM_ACTION.toUpperCase()} for ${params.ENVIRONMENT}?",
                          ok: "✅ Proceed"
                }
            }
        }

        stage('Apply Infrastructure') {
            when {
                expression { params.TERRAFORM_ACTION == 'apply' }
            }
            steps {
                dir("${params.TERRAFORM_DIR}") {
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        echo "🚀 Applying Terraform for ${params.ENVIRONMENT} in ${params.AWS_DEFAULT_REGION}"
                        sh """
                            set -e
                            export AWS_DEFAULT_REGION=${params.AWS_DEFAULT_REGION}
                            terraform apply -auto-approve -var-file=${params.ENVIRONMENT}.tfvars -var="region=${params.AWS_DEFAULT_REGION}" ${env.TF_PLAN_FILE}
                        """
                    }
                }
            }
        }

        stage('Destroy Infrastructure') {
            when {
                expression { params.TERRAFORM_ACTION == 'destroy' }
            }
            steps {
                dir("${params.TERRAFORM_DIR}") {
                    withCredentials([
                        string(credentialsId: 'AWS_ACCESS_KEY_ID', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'AWS_SECRET_ACCESS_KEY', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        echo "💣 Destroying Terraform resources for ${params.ENVIRONMENT} in ${params.AWS_DEFAULT_REGION}"
                        sh """
                            set -e
                            export AWS_DEFAULT_REGION=${params.AWS_DEFAULT_REGION}
                            terraform destroy -auto-approve -var-file=${params.ENVIRONMENT}.tfvars -var="region=${params.AWS_DEFAULT_REGION}"
                        """
                    }
                }
            }
        }

        stage('Completed') {
            steps {
                echo """
🎉 Terraform ${params.TERRAFORM_ACTION.toUpperCase()} completed successfully!
📦 Environment : ${params.ENVIRONMENT}
🌍 AWS Region   : ${params.AWS_DEFAULT_REGION}
🪣 Repo         : ${params.TERRAFORM_REPO}
🌿 Branch       : ${params.TERRAFORM_BRANCH}
📁 Directory    : ${params.TERRAFORM_DIR}
                """
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline executed successfully for ${params.ENVIRONMENT} (${params.AWS_DEFAULT_REGION})"
        }
        failure {
            echo "❌ Pipeline failed for ${params.ENVIRONMENT} (${params.AWS_DEFAULT_REGION})"
        }
        always {
            echo "📘 Jenkins Terraform pipeline finished"
        }
    }
}
