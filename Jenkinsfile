pipeline {
    agent any

    options {
        timestamps()
    }

    parameters {

        choice(
            name: 'DEPLOY_FORMAT',
            choices: ['SOURCE', 'MDAPI'],
            description: 'Deployment format'
        )

        choice(
            name: 'DEPLOY_SCOPE',
            choices: ['FULL', 'DELTA'],
            description: 'Deployment scope'
        )

        choice(
            name: 'TEST_LEVEL',
            choices: ['NoTestRun', 'RunLocalTests'],
            description: 'Salesforce test level'
        )

        string(
            name: 'DELTA_BASELINE',
            defaultValue: 'origin/main',
            description: 'Git ref for delta baseline'
        )

        string(
            name: 'PACKAGE_XML_PATH',
            defaultValue: 'manifest/package.xml',
            description: 'package.xml path (MDAPI only)'
        )

        string(
            name: 'APEX_CLASSES',
            defaultValue: '',
            description: 'Comma-separated Apex classes (optional)'
        )
    }

    environment {
        SF              = "/c/Program Files/sf/bin/sf"
        SF_ALIAS        = "CICD_QAHub"
        INSTANCE_URL    = "https://login.salesforce.com"
        SF_USERNAME     = "manisha.mondal@accenture.com"
        DELTA_DIR       = "delta"
    }

    stages {

        /* ---------------------------------------------------------
           Checkout
           --------------------------------------------------------- */
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/manishamondal/salesforce-full.git'
                    ]]
                ])
            }
        }

        /* ---------------------------------------------------------
           Verify Git
           --------------------------------------------------------- */
        stage('Verify Git') {
            steps {
                sh 'git branch -a'
                sh 'git log -1'
            }
        }

        /* ---------------------------------------------------------
           Salesforce Auth (JWT)
           --------------------------------------------------------- */
        stage('Authenticate to Salesforce') {
            steps {
                withCredentials([
                    file(credentialsId: 'sfdx_jwt_key', variable: 'JWT_KEY_FILE'),
                    string(credentialsId: 'sfdx_client_id', variable: 'CLIENT_ID')
                ]) {
                    sh """
                    "$SF" org login jwt \
                      --client-id "$CLIENT_ID" \
                      --jwt-key-file "$JWT_KEY_FILE" \
                      --username "$SF_USERNAME" \
                      --instance-url "$INSTANCE_URL" \
                      --alias "$SF_ALIAS" \
                      --set-default
                    """
                }
            }
        }

        /* ---------------------------------------------------------
           Install Plugins
           --------------------------------------------------------- */
        stage('Install Plugins') {
            steps {
                sh '''
                echo y | "$SF" plugins install sfdx-git-delta || true
                "$SF" plugins install @salesforce/sfdx-scanner || true
                '''
            }
        }

        /* ---------------------------------------------------------
           Validate Delta Baseline
           --------------------------------------------------------- */
        stage('Validate Delta Baseline') {
            when {
                expression { params.DEPLOY_SCOPE == 'DELTA' }
            }
            steps {
                sh """
                echo "Validating baseline: ${DELTA_BASELINE}"
                git rev-parse --verify ${DELTA_BASELINE}
                """
            }
        }

        /* ---------------------------------------------------------
           Generate Delta
           --------------------------------------------------------- */
        stage('Generate Delta') {
            when {
                expression {
                    params.DEPLOY_SCOPE == 'DELTA' &&
                    params.APEX_CLASSES.trim() == ''
                }
            }
            steps {
                sh """
                rm -rf "$DELTA_DIR"
                mkdir -p "$DELTA_DIR"

                "$SF" sgd:source:delta \
                  --from ${DELTA_BASELINE} \
                  --to HEAD \
                  --output "$DELTA_DIR" \
                  --generate-delta
                """
            }
        }

        /* ---------------------------------------------------------
           Static Code Analysis (Delta only)
           --------------------------------------------------------- */
        stage('Static Code Analysis') {
            when {
                expression { params.DEPLOY_SCOPE == 'DELTA' }
            }
            steps {
                sh """
                mkdir -p /tmp/SCA
                "$SF" code-analyzer run \
                  --workspace "$DELTA_DIR" \
                  --rule-selector Recommended \
                  --output-file /tmp/SCA/sca-report.html
                """
            }
        }

        /* ---------------------------------------------------------
           Validate Deployment (Dry Run)
           --------------------------------------------------------- */
        stage('Validate Deployment') {
            steps {
                script {

                    if (params.APEX_CLASSES.trim()) {
                        def classPaths = params.APEX_CLASSES
                            .split(',')
                            .collect { it.trim() }
                            .collect {
                                "force-app/main/default/classes/${it}.cls," +
                                "force-app/main/default/classes/${it}.cls-meta.xml"
                            }
                            .join(',')

                        sh """
                        rm -rf .sf .sfdx
                        "$SF" deploy metadata \
                          --source-dir ${classPaths} \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --ignore-conflicts \
                          --dry-run \
                          --wait 60
                        """
                        return
                    }

                    if (params.DEPLOY_FORMAT == 'MDAPI') {
                        sh """
                        rm -rf .sf .sfdx
                        "$SF" deploy metadata \
                          --manifest ${params.PACKAGE_XML_PATH} \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --dry-run \
                          --wait 60
                        """
                        return
                    }

                    if (params.DEPLOY_SCOPE == 'DELTA') {
                        sh """
                        rm -rf .sf .sfdx
                        "$SF" deploy metadata \
                          --source-dir "$DELTA_DIR" \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --dry-run \
                          --wait 60
                        """
                    } else {
                        sh """
                        rm -rf .sf .sfdx
                        "$SF" deploy metadata \
                          --source-dir force-app \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --dry-run \
                          --wait 60
                        """
                    }
                }
            }
        }

        /* ---------------------------------------------------------
           Manual Approval
           --------------------------------------------------------- */
        stage('Approve Deployment') {
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: """
Dry-run validation successful.

Deployment details:
- Format : ${params.DEPLOY_FORMAT}
- Scope  : ${params.DEPLOY_SCOPE}
- Baseline : ${params.DELTA_BASELINE}
- Apex-only : ${params.APEX_CLASSES ?: 'No'}

Approve deployment?
""",
                    ok: 'Approve'
                }
            }
        }

        /* ---------------------------------------------------------
           Deploy (Actual)
           --------------------------------------------------------- */
        stage('Deploy') {
            steps {
                script {

                    if (params.APEX_CLASSES.trim()) {
                        def classPaths = params.APEX_CLASSES
                            .split(',')
                            .collect { it.trim() }
                            .collect {
                                "force-app/main/default/classes/${it}.cls," +
                                "force-app/main/default/classes/${it}.cls-meta.xml"
                            }
                            .join(',')

                        sh """
                        rm -rf .sf .sfdx
                        "$SF" deploy metadata \
                          --source-dir ${classPaths} \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --wait 60
                        """
                        return
                    }

                    if (params.DEPLOY_FORMAT == 'MDAPI') {
                        sh """
                        rm -rf .sf .sfdx
                        "$SF" deploy metadata \
                          --manifest ${params.PACKAGE_XML_PATH} \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --wait 60
                        """
                        return
                    }

                    if (params.DEPLOY_SCOPE == 'DELTA') {
                        sh """
                        rm -rf .sf .sfdx
                        "$SF" deploy metadata \
                          --source-dir "$DELTA_DIR" \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --wait 60
                        """
                    } else {
                        sh """
                        rm -rf .sf .sfdx
                        "$SF" deploy metadata \
                          --source-dir force-app \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --wait 60
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo '✅ Salesforce deployment completed successfully'
        }
        failure {
            echo '❌ Salesforce pipeline failed'
        }
        aborted {
            echo '⚠ Deployment aborted or approval expired'
        }
    }
}
