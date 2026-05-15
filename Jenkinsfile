pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        STACK_NAME        = 'todo-list-aws-production'
    }

    stages {
        stage('Get Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/ericmatamoros/unir-todo-list-aws'
                sh '''
                    curl -fsSL \
                        https://raw.githubusercontent.com/ericmatamoros/todo-list-aws-config/production/samconfig.toml \
                        -o samconfig.toml
                    echo "--- samconfig.toml descargado (production) ---"
                    cat samconfig.toml
                '''
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    sam build
                    sam validate --region $AWS_DEFAULT_REGION
                    sam deploy \
                        --config-env production \
                        --resolve-s3 \
                        --no-confirm-changeset \
                        --no-fail-on-empty-changeset
                '''
            }
        }

        stage('Rest Test') {
            steps {
                script {
                    env.BASE_URL = sh(
                        script: """
                            aws cloudformation describe-stacks \
                                --stack-name ${STACK_NAME} \
                                --region ${AWS_DEFAULT_REGION} \
                                --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                                --output text
                        """,
                        returnStdout: true
                    ).trim()
                    echo "BASE_URL = ${env.BASE_URL}"
                }
                sh '''
                    export PYTHONPATH=$WORKSPACE
                    python3 -m pytest --junitxml=result-rest.xml \
                        test/integration/todoApiTest.py \
                        -k "test_api_listtodos or test_api_gettodo"
                '''
                junit 'result-rest.xml'
            }
        }
    }
}
