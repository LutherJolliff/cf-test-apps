pipeline {
    environment {
        TF_VAR_eb_app_name = 'sampleApp'
        TF_VAR_role_arn = credentials('tf-role-arn')
        TF_VAR_cookie_encrypt_pass = credentials('sampleApp-cookie-encrypt')
        TF_VAR_sql_user = credentials('sampleApp-sql-user')
        TF_VAR_sql_password = credentials('sampleApp-sql-password')
        TF_VAR_sql_database = credentials('sampleApp-sql-database')
        AWS_ACCESS_KEY_ID = credentials('tf_aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('tf_secret_access_key_id')
    }

    // agent {
    //     docker {
    //         image 'luther007/cynerge_images:latest'
    //         args '-u root'
    //     }
    // }
    agent any

    stages {
        stage('dependencies') {
            steps {
                echo 'Installing...'
                sh 'npm ci'
            }
        }
        stage('test') {
            steps {
                echo 'Testing...'
                sh 'npm run test'
            }
        }
        stage('clone-iaas-repo') {
            steps {
                sh 'rm terraform -rf; mkdir terraform'
                dir ('terraform') {
                    git branch: 'cf_js_database_sqlServer',
                        credentialsId: 'luther-github-ssh',
                        url: 'git@github.com:cynerge-consulting/non-containerized-pipeline-tf.git'
                }
            }
        }
        stage('provision-infrastructure') {
            when {
                anyOf {
                    branch 'js_db_sqlServer'
                }
            }
            steps {
                script {
                    sh 'ls'
                    sh '''
                        terraform --version
                        terraform init
                        terraform plan -input=false
                        terraform apply --auto-approve
                    '''
                }
            }
        }
        stage('dev-deploy') {
            when {
                branch 'js_db_sqlServer'
            }
            agent {
                docker {
                    image 'cynergeconsulting/aws-cli:latest'
                    args '-u root'
                }
            }
            steps {
                // sh 'cd '
                sh 'ls'
                sh 'env | grep AWS'
                sh 'aws s3 ls'
                sh 'eb deploy development'
                sh 'sleep 5'
                sh 'aws elasticbeanstalk describe-environments --environment-names development --query "Environments[*].CNAME" --output text'
            }
        }
        stage('staging-deploy') {
            when {
                branch 'staging'
            }
            steps {
                sh '''
                    echo 'export PATH="/root/.ebcli-virtual-env/executables:$PATH"' >> ~/.bash_profile
                    eb deploy staging
                    sleep 5
                    aws elasticbeanstalk describe-environments --environment-names staging --query "Environments[*].CNAME" --output text
                '''
            }
        }
        stage('prod-deploy') {
            when {
                branch 'prod'
            }
            steps {
                sh '''
                    echo 'export PATH="/root/.ebcli-virtual-env/executables:$PATH"' >> ~/.bash_profile
                    eb deploy production
                    sleep 5
                    aws elasticbeanstalk describe-environments --environment-names production --query "Environments[*].CNAME" --output text
                '''
            }
        }
    }
}