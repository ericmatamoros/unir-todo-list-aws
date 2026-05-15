pipeline {
    agent any

    environment {
        AWS_DEFAULT_REGION = 'us-east-1'
        STACK_NAME        = 'todo-list-aws-staging'
    }

    stages {
        stage('Get Code') {
            steps {
                git branch: 'develop',
                    url: 'https://github.com/ericmatamoros/unir-todo-list-aws'
                sh '''
                    curl -fsSL \
                        https://raw.githubusercontent.com/ericmatamoros/todo-list-aws-config/staging/samconfig.toml \
                        -o samconfig.toml
                    echo "--- samconfig.toml descargado (staging) ---"
                    cat samconfig.toml
                '''
            }
        }

        stage('Static Test') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh '''
                        export PYTHONPATH=$WORKSPACE
                        export DYNAMODB_TABLE=test-TodosDynamoDbTable
                        export ENDPOINT_OVERRIDE=""
                        python3 -m pytest --junitxml=result-unit.xml test/unit
                    '''
                }
                junit 'result-unit.xml'

                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sh 'flake8 --format=pylint src > flake8.out || true'
                }
                catchError(buildResult: 'SUCCESS', stageResult: 'SUCCESS') {
                    sh 'bandit -r src -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}" || true'
                }
                recordIssues(
                    tools: [
                        flake8(name: 'Flake8', pattern: 'flake8.out'),
                        pyLint(name: 'Bandit', pattern: 'bandit.out')
                    ]
                )
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    sam build
                    sam validate --region $AWS_DEFAULT_REGION
                    sam deploy \
                        --config-env staging \
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
                    python3 -m pytest --junitxml=result-rest.xml test/integration/todoApiTest.py
                '''
                junit 'result-rest.xml'
            }
        }

        stage('Promote') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-token',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh '''
                        git config user.email "jenkins@ci.local"
                        git config user.name  "Jenkins CI"
                        git fetch origin
                        git checkout master
                        git pull origin master
                        git merge --no-ff --no-commit origin/develop || true
                        # Preserve master's CD pipeline files: they must not be overwritten by develop's CI versions
                        for f in Jenkinsfile Jenkinsfile_agentes; do
                            if git show HEAD:$f >/dev/null 2>&1; then
                                git show HEAD:$f > $f
                                git add $f
                            fi
                        done
                        git commit -m "Promote develop to master [CI]"
                        git push "https://${GIT_USER}:${GIT_TOKEN}@github.com/ericmatamoros/unir-todo-list-aws.git" master
                    '''
                }
            }
        }
    }
}
