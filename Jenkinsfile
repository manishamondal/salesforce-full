pipeline {
    agent any

    options {
        timestamps()
    }

    environment {
        REPO_URL     = 'https://github.com/manishamondal/salesforce-full.git'
        BRANCH_NAME  = 'main'
        BASELINE_TAG = 'baseline-last-success'
        DELTA_DIR    = 'delta'
        SF_CLI       = '/c/Program Files/sf/bin/sf'
    }

    stages {

        stage('Checkout') {
            steps {
                echo '📥 Checking out source code (main branch)'

                // Explicit refspec to avoid Jenkins defaulting to master
                git(
                    url: "${REPO_URL}",
                    branch: "${BRANCH_NAME}",
                    changelog: false,
                    poll: false
                )

                sh 'git branch'
            }
        }

        stage('Capture Git Commit') {
            steps {
                script {
                    env.CURRENT_COMMIT = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    echo "📌 Current Git Commit: ${env.CURRENT_COMMIT}"
                }
            }
        }

        stage('Resolve Baseline') {
            steps {
                script {
                    def tagExists = sh(
                        script: "git tag -l ${BASELINE_TAG}",
                        returnStdout: true
                    ).trim()

                    if (tagExists) {
                        env.BASELINE_COMMIT = sh(
                            script: "git rev-list -n 1 ${BASELINE_TAG}",
                            returnStdout: true
                        ).trim()
                        echo "🔖 Baseline found at commit: ${env.BASELINE_COMMIT}"
                    } else {
                        env.BASELINE_COMMIT = env.CURRENT_COMMIT
                        echo "⚠️ Baseline tag not found (first run)"
                        echo "➡️ Treating this as FULL deployment"
                    }
                }
            }
        }

        stage('Authenticate to Salesforce') {
            steps {
                withCredentials([
                    file(credentialsId: 'sfdx_jwt_key', variable: 'JWT_KEY_FILE'),
                    string(credentialsId: "${env.SF_CLIENT_ID_CRED}", variable: 'CLIENT_ID'),
                    string(credentialsId: "${env.SF_USERNAME_CRED}", variable: 'SF_USERNAME')
                ]) {
                     sh '''
                    "$SF" org login jwt \
                      --client-id "$CLIENT_ID" \
                      --jwt-key-file "$JWT_KEY_FILE" \
                      --username "$SF_USERNAME" \
                      --instance-url "$INSTANCE_URL" \
                      --alias "$SF_ALIAS" \
                      --set-default
                    '''
                }
            }
        }

        stage('Generate Delta') {
            steps {
                script {
                    sh "mkdir -p ${DELTA_DIR}"

                    if (env.BASELINE_COMMIT == env.CURRENT_COMMIT) {
                        echo "📦 FULL deployment (initial run)"
                        sh "cp -r force-app ${DELTA_DIR}/force-app"
                    } else {
                        echo "🔄 Generating DELTA package"
                        sh """
                            "${SF_CLI}" sgd:source:delta \
                              --from ${env.BASELINE_COMMIT} \
                              --to ${env.CURRENT_COMMIT} \
                              --output-dir ${DELTA_DIR} \
                              --source-dir force-app
                        """
                    }
                }
            }
        }

        stage('Validate Deployment') {
            steps {
                echo '🧪 Validating deployment'
                sh """
                    "${SF_CLI}" deploy validate source \
                      --source-dir ${DELTA_DIR}/force-app \
                      --target-org CICD_DevHub \
                      --test-level NoTestRun \
                      --wait 60
                """
            }
        }

        stage('Deploy') {
            steps {
                echo '🚀 Deploying to Salesforce'
                sh """
                    "${SF_CLI}" deploy start source \
                      --source-dir ${DELTA_DIR}/force-app \
                      --target-org CICD_DevHub \
                      --test-level NoTestRun \
                      --wait 60
                """
            }
        }

        stage('Update Baseline Tag') {
            steps {
                script {
                    echo '🔖 Updating baseline tag'
                    sh """
                        git tag -f ${BASELINE_TAG} ${CURRENT_COMMIT}
                        git push origin ${BASELINE_TAG} --force
                    """
                }
            }
        }
    }

    post {
        success {
            echo '✅ SALESFORCE PIPELINE COMPLETED SUCCESSFULLY'
        }
        failure {
            echo '❌ SALESFORCE PIPELINE FAILED'
        }
    }
}
