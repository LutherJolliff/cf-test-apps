pipeline {
    environment {
        TF_VAR_eb_app_name = 'sampleapp'
        TF_VAR_role_arn = credentials('tf-role-arn')
        TF_VAR_cookie_encrypt_pass = credentials('sampleApp-cookie-encrypt')
        TF_VAR_sql_user = credentials('sampleApp-sql-user')
        TF_VAR_sql_password = credentials('sampleApp-sql-password')
        TF_VAR_sql_database = credentials('sampleApp-sql-database')
        AWS_ACCESS_KEY_ID = credentials('tf_aws_access_key_id')
        AWS_SECRET_ACCESS_KEY = credentials('tf_secret_access_key_id')
        AWS_REGION = 'us-gov-west-1'
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
                sh 'which docker'
                sh "echo $PATH"
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
                    dir ('terraform') {
                        sh '''
                            terraform --version
                            terraform init -backend-config "bucket=fs-${TF_VAR_eb_app_name}-nonprod"
                            terraform plan -input=false
                            terraform apply --auto-approve
                        '''
                    }
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
                sh '''
                    VERSION=$( date '+%F_%H:%M:%S' )
                    zip -r ${BUILD_TAG}.zip .
                    aws s3 cp ./$BUILD_TAG.zip s3://fs-${TF_VAR_eb_app_name}-nonprod/$TF_VAR_eb_app_name/
                    aws elasticbeanstalk create-application-version --application-name $TF_VAR_eb_app_name --version-label v${BUILD_NUMBER}_${VERSION} --description="Built by Jenkins job $JOB_NAME" --source-bundle S3Bucket="fs-${TF_VAR_eb_app_name}-nonprod",S3Key="$TF_VAR_eb_app_name/${BUILD_TAG}.zip" --region=$AWS_REGION
                    sleep 2
                    aws elasticbeanstalk update-environment --application-name $TF_VAR_eb_app_name --environment-name development --version-label v${BUILD_NUMBER}_${VERSION} --region=$AWS_REGION
                '''
            }
        }
        stage('staging-deploy') {
            when {
                branch 'staging'
            }
            steps {
                sh '''
                    VERSION=$( date '+%F_%H:%M:%S' )
                    zip -r ${BUILD_TAG}.zip .
                    aws s3 cp ./$BUILD_TAG.zip s3://fs-${TF_VAR_eb_app_name}-nonprod/$TF_VAR_eb_app_name/
                    aws elasticbeanstalk create-application-version --application-name $TF_VAR_eb_app_name --version-label v${BUILD_NUMBER}_${VERSION} --description="Built by Jenkins job $JOB_NAME" --source-bundle S3Bucket="fs-${TF_VAR_eb_app_name}-nonprod",S3Key="$TF_VAR_eb_app_name/${BUILD_TAG}.zip" --region=$AWS_REGION
                    sleep 2
                    aws elasticbeanstalk update-environment --application-name $TF_VAR_eb_app_name --environment-name development --version-label v${BUILD_NUMBER}_${VERSION} --region=$AWS_REGION
                '''
            }
        }
        stage('prod-deploy') {
            when {
                branch 'prod'
            }
            steps {
                sh '''
                    VERSION=$( date '+%F_%H:%M:%S' )
                    zip -r ${BUILD_TAG}.zip .
                    aws s3 cp ./$BUILD_TAG.zip s3://fs-${TF_VAR_eb_app_name}-prod/$TF_VAR_eb_app_name/
                    aws elasticbeanstalk create-application-version --application-name $TF_VAR_eb_app_name --version-label v${BUILD_NUMBER}_${VERSION} --description="Built by Jenkins job $JOB_NAME" --source-bundle S3Bucket="fs-${TF_VAR_eb_app_name}-prod",S3Key="$TF_VAR_eb_app_name/${BUILD_TAG}.zip" --region=$AWS_REGION
                    sleep 2
                    aws elasticbeanstalk update-environment --application-name $TF_VAR_eb_app_name --environment-name development --version-label v${BUILD_NUMBER}_${VERSION} --region=$AWS_REGION
                '''
            }
        }
    }
}