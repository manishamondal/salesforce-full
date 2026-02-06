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

        SF           = '/c/Program Files/sf/bin/sf'
        INSTANCE_URL = 'https://login.salesforce.com'
        SF_ALIAS     = 'CICD_DevHub'
    }

    stages {

        stage('Checkout') {
            steps {
                echo '📥 Checking out source code (main branch)'
                git branch: BRANCH_NAME, url: REPO_URL
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
                        echo "🔖 Baseline found: ${env.BASELINE_COMMIT}"
                    } else {
                        env.BASELINE_COMMIT = env.CURRENT_COMMIT
                        echo "⚠️ Baseline not found → FULL deployment"
                    }
                }
            }
        }

        stage('Authenticate to Salesforce') {
            steps {
                withCredentials([
                    file(credentialsId: 'sfdx_jwt_key', variable: 'JWT_KEY_FILE'),
                    string(credentialsId: 'sfdx_client_id', variable: 'CLIENT_ID'),
                    string(credentialsId: 'sfdx_username', variable: 'SF_USERNAME')
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
                        echo "📦 FULL deployment"
                        sh "cp -r force-app ${DELTA_DIR}/force-app"
                    } else {
                        echo "🔄 DELTA deployment"
                        sh """
                        "$SF" sgd:source:delta \
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
                sh """
                "$SF" project deploy start \
                  --source-dir ${DELTA_DIR}/force-app \
                  --target-org "$SF_ALIAS" \
                  --test-level NoTestRun \
                  --dry-run \
                  --wait 60
                """
            }
        }

        stage('Deploy') {
            steps {
                sh """
                "$SF" project deploy start \
                  --source-dir ${DELTA_DIR}/force-app \
                  --target-org "$SF_ALIAS" \
                  --test-level NoTestRun \
                  --wait 60
                """
            }
        }

        stage('Update Baseline Tag') {
            steps {
                sh """
                git tag -f ${BASELINE_TAG} ${CURRENT_COMMIT}
                git push origin ${BASELINE_TAG} --force
                """
            }
        }
    }

    post {
        success {
            echo '✅ SALESFORCE DELTA PIPELINE SUCCESSFUL'
        }
        failure {
            echo '❌ SALESFORCE DELTA PIPELINE FAILED'
        }
    }
}
