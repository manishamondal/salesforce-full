pipeline {
    agent any

    options {
        timestamps()
    }

    parameters {

        /* ===================== NEW ===================== */
        choice(
            name: 'TARGET_ENV',
            choices: ['DEV', 'QA'],
            description: 'Target Salesforce environment'
        )
        /* ================================================= */

        choice(
            name: 'DEPLOY_FORMAT',
            choices: ['SOURCE', 'MDAPI'],
            description: 'Deployment format'
        )

        choice(
            name: 'DEPLOY_SCOPE',
            choices: ['FULL', 'DELTA'],
            description: 'Deployment scope - FULL or DELTA (from baseline tag)'
        )

        choice(
            name: 'TEST_LEVEL',
            choices: ['NoTestRun', 'RunLocalTests'],
            description: 'Salesforce test level'
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

        booleanParam(
            name: 'ROLLBACK_MODE',
            defaultValue: false,
            description: 'Enable rollback to previous successful build'
        )

        string(
            name: 'ROLLBACK_TO_BUILD',
            defaultValue: '',
            description: 'Build number to rollback to (leave empty to select interactively)'
        )
    }

    environment {
        SF              = "/c/Program Files/sf/bin/sf"
        INSTANCE_URL    = "https://login.salesforce.com"

        DELTA_DIR       = "delta"
        SCA_DIR         = "sca-reports"
        SCA_THRESHOLD_HIGH = "0"
        SCA_THRESHOLD_CRITICAL = "0"

        DEPLOYMENT_ID   = ""
        BUILD_METRICS   = "build-metrics.json"

        CLEAN_BUILD_STREAK = "0"
        PREVIOUS_BUILD_CLEAN = "false"
        GIT_COMMIT_HASH = ""
           // ✅ BASELINE TAG (ENV AWARE)
        BASELINE_TAG = "baseline-${TARGET_ENV}"
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

stage('Capture Git Commit') {
    steps {
        script {
            def commit = sh(
                script: "git rev-parse HEAD",
                returnStdout: true
            ).trim()

            sh """
              mkdir -p logs
              echo ${commit} > logs/git-commit.txt
            """

            echo "📌 Captured Git Commit: ${commit}"
        }
    }
}


        /* ---------------------------------------------------------
           Verify Git
           --------------------------------------------------------- */
        stage('Verify Git') {
            steps {
                script {
                    sh 'git branch -a'
                    sh 'git log -1'
                    
                    // Capture current commit hash for tracking
                    env.GIT_COMMIT_HASH = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()
                     sh """
                    mkdir -p logs
                    echo ${env.GIT_COMMIT_HASH} > logs/git-commit.txt
                    """
                    echo "Current Git Commit: ${env.GIT_COMMIT_HASH}"
                }
            }
        }

        /* ---------------------------------------------------------
           Rollback Handler
           --------------------------------------------------------- */
        stage('Rollback') {
            when {
                expression { params.ROLLBACK_MODE == true }
            }
            steps {
                script {
                    echo ''
                    echo '🔄 ========================================'
                    echo '🔄  ROLLBACK MODE ACTIVATED'
                    echo '🔄 ========================================'
                    echo ''
                    
                    def targetBuildNumber = params.ROLLBACK_TO_BUILD
                    
                    // If no build number specified, list recent successful builds
                    if (!targetBuildNumber || targetBuildNumber.trim() == '') {
                        echo "Fetching recent successful builds..."
                        
                        def recentBuilds = []
                        def build = currentBuild.previousSuccessfulBuild
                        def count = 0
                        
                        while (build != null && count < 10) {
                            try {
                                def buildArchive = "${JENKINS_HOME}/jobs/${JOB_NAME}/builds/${build.number}/archive"
                                def gitHashFile = "${buildArchive}/logs/git-commit.txt"
                                
                                def gitHashExists = sh(
                                    script: "test -f '${gitHashFile}' && echo 'true' || echo 'false'",
                                    returnStdout: true
                                ).trim()
                                
                                def gitHash = 'unknown'
                                if (gitHashExists == 'true') {
                                    gitHash = sh(
                                        script: "cat '${gitHashFile}' | head -n 1",
                                        returnStdout: true
                                    ).trim()
                                }
                                
                                recentBuilds.add("Build #${build.number} (${new Date(build.startTimeInMillis).format('yyyy-MM-dd HH:mm')} - Commit: ${gitHash.take(8)})")
                            } catch (Exception e) {
                                recentBuilds.add("Build #${build.number} (${new Date(build.startTimeInMillis).format('yyyy-MM-dd HH:mm')})")
                            }
                            
                            build = build.previousSuccessfulBuild
                            count++
                        }
                        
                        if (recentBuilds.isEmpty()) {
                            error('❌ No previous successful builds found for rollback')
                        }
                        
                        echo "Available builds for rollback:"
                        recentBuilds.each { echo "  - ${it}" }
                        
                        def buildChoice = input(
                            message: 'Select build to rollback to:',
                            parameters: [
                                choice(
                                    name: 'BUILD_SELECTION',
                                    choices: recentBuilds,
                                    description: 'Choose a previous successful build'
                                )
                            ]
                        )
                        
                        // Extract build number from selection
                        targetBuildNumber = buildChoice.replaceAll(/.*Build #(\d+).*/, '$1')
                        echo "Selected Build: #${targetBuildNumber}"
                    }
                    
                    // Retrieve Git commit hash from target build
                    def targetBuildArchive = "${JENKINS_HOME}/jobs/${JOB_NAME}/builds/${targetBuildNumber}/archive"
                    def gitHashFile = "${targetBuildArchive}/logs/git-commit.txt"
                    
                    def gitHashExists = sh(
                        script: "test -f '${gitHashFile}' && echo 'true' || echo 'false'",
                        returnStdout: true
                    ).trim()
                    
                    def targetGitHash = ''
                    
                    if (gitHashExists == 'true') {
                        targetGitHash = sh(
                            script: "cat '${gitHashFile}' | head -n 1",
                            returnStdout: true
                        ).trim()
                    } else {
                        echo "⚠️  Warning: Git commit hash file not found for Build #${targetBuildNumber}"
                        echo "This build was created before rollback feature was implemented."
                        echo ""
                        
                        // Prompt user to manually enter git commit hash
                        targetGitHash = input(
                            message: "Enter the Git commit hash to rollback to:",
                            parameters: [
                                string(
                                    name: 'COMMIT_HASH',
                                    defaultValue: '',
                                    description: 'Git commit SHA (find it in build logs or Git history)'
                                )
                            ]
                        )
                        
                        if (!targetGitHash || targetGitHash.trim().isEmpty()) {
                            error("❌ No Git commit hash provided. Cannot proceed with rollback.")
                        }
                        
                        targetGitHash = targetGitHash.trim()
                    }
                    
                    echo ''
                    echo '📋 Rollback Details:'
                    echo "   Target Build: #${targetBuildNumber}"
                    echo "   Git Commit: ${targetGitHash}"
                    echo "   Current Commit: ${env.GIT_COMMIT_HASH}"
                    echo ''
                    
                    // Final confirmation
                    input(
                        message: """⚠️  CONFIRM ROLLBACK

You are about to rollback to Build #${targetBuildNumber}
Git Commit: ${targetGitHash}

This will:
1. Checkout the previous commit
2. Redeploy metadata to Salesforce org
3. Archive this rollback as a new build

Current deployment will be REPLACED.

Proceed with rollback?""",
                        ok: 'Yes, Rollback Now'
                    )
                    
                    // Checkout target commit
                    echo "🔄 Checking out commit ${targetGitHash}..."
                    sh """
                    git fetch --all
                    git checkout ${targetGitHash}
                    git log -1
                    """
                    
                    // Update environment variable
                    env.GIT_COMMIT_HASH = targetGitHash
                    
                    echo '✅ Successfully checked out target commit'
                    echo '📦 Proceeding with deployment of rolled-back code...'
                    echo ''
                }
            }
        }

        /* ---------------------------------------------------------
           Check Previous Build SCA (UNCHANGED)
           --------------------------------------------------------- */
        stage('Check SCA History') {
            steps {
                script {
                    echo "=== Checking Previous Build SCA Results ==="

                    def currentStreak = 0
                    def previousBuildClean = false

                    try {
                        def previousBuild = currentBuild.previousSuccessfulBuild

                        if (previousBuild != null) {
                            def previousWorkspace =
                                "${JENKINS_HOME}/jobs/${JOB_NAME}/builds/${previousBuild.number}/archive"

                            def scaSummaryExists = sh(
                                script: "test -f '${previousWorkspace}/sca-reports/summary.txt' && echo true || echo false",
                                returnStdout: true
                            ).trim()

                            if (scaSummaryExists == 'true') {
                                def scaSummary = sh(
                                    script: "cat '${previousWorkspace}/sca-reports/summary.txt'",
                                    returnStdout: true
                                ).trim()

                                if (scaSummary.contains('✅ PASSED')) {
                                    previousBuildClean = true

                                    def streakFileExists = sh(
                                        script: "test -f '${previousWorkspace}/logs/sca-streak.txt' && echo true || echo false",
                                        returnStdout: true
                                    ).trim()

                                    if (streakFileExists == 'true') {
                                        currentStreak = sh(
                                            script: "cat '${previousWorkspace}/logs/sca-streak.txt'",
                                            returnStdout: true
                                        ).trim().toInteger()
                                    }
                                }
                            }
                        }

                        env.CLEAN_BUILD_STREAK = currentStreak.toString()
                        env.PREVIOUS_BUILD_CLEAN = previousBuildClean.toString()

                    } catch (e) {
                        env.CLEAN_BUILD_STREAK = "0"
                        env.PREVIOUS_BUILD_CLEAN = "false"
                    }
                }
            }
        }

        /* =========================================================
           NEW STAGE – Environment Configuration
           ========================================================= */
        stage('Configure Target Environment') {
            steps {
                script {
                    echo "Configuring environment for ${params.TARGET_ENV}"

                    if (params.TARGET_ENV == 'DEV') {
                        env.SF_ALIAS = 'CICD_DevHub'
                        env.SF_USERNAME_CRED = 'sfdx_username'
                        env.SF_CLIENT_ID_CRED = 'sfdx_client_id'
                    } else if (params.TARGET_ENV == 'QA') {
                        env.SF_ALIAS = 'CICD_QA'
                        env.SF_USERNAME_CRED = 'sfdx_username_qa'
                        env.SF_CLIENT_ID_CRED = 'sfdx_client_id_qa'
                    }

                    echo """
Target Env  : ${params.TARGET_ENV}
SF Alias    : ${env.SF_ALIAS}
User Cred   : ${env.SF_USERNAME_CRED}
Client Cred : ${env.SF_CLIENT_ID_CRED}
"""
                }
            }
        }

        /* ---------------------------------------------------------
           Salesforce Auth (JWT) – UPDATED ONLY HERE
           --------------------------------------------------------- */
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

        /* ---------------------------------------------------------
           Install Plugins
           --------------------------------------------------------- */
        // stage('Install Plugins') {
        //     steps {
        //         sh '''
        //         echo y | "$SF" plugins install sfdx-git-delta || true
        //         "$SF" plugins install @salesforce/sfdx-scanner || true
        //         '''
        //     }
        // }

        /* ---------------------------------------------------------
           Static Code Analysis (ALL Deployments)
           --------------------------------------------------------- */
        stage('Static Code Analysis') {
            steps {
                script {
                    def scanDir = 'force-app'
                    
                    if (params.APEX_CLASSES.trim()) {
                        scanDir = 'force-app/main/default/classes'
                    }
                    
                    sh """
                    mkdir -p "$SCA_DIR"
                    
                    echo "=== Running Salesforce Code Analyzer ==="
                    echo "Scan Directory: ${scanDir}"
                    echo "Timestamp: \$(date)"
                    
                    # Run code analyzer with multiple formats
                    "$SF" scanner run \
                      --target "${scanDir}" \
                      --format json  \
                      --outfile "$SCA_DIR/sca-report.json" \
                      --severity-threshold 1 \
                      --normalize-severity || true
                    
                    "$SF" scanner run \
                      --target "${scanDir}" \
                      --format csv  \
                      --outfile "$SCA_DIR/sca-report.csv" \
                      --severity-threshold 1 \
                      --normalize-severity || true                  

                    "$SF" scanner run \
                      --target "${scanDir}" \
                      --format html \
                      --outfile "$SCA_DIR/sca-report.html" \
                      --severity-threshold 1 \
                      --normalize-severity || true
                    
                    # Generate summary report
                    echo "=== Static Code Analysis Summary ===" > "$SCA_DIR/summary.txt"
                    echo "Build: ${BUILD_NUMBER}" >> "$SCA_DIR/summary.txt"
                    echo "Date: \$(date)" >> "$SCA_DIR/summary.txt"
                    echo "Scope: ${params.DEPLOY_SCOPE}" >> "$SCA_DIR/summary.txt"
                    echo "Directory Scanned: ${scanDir}" >> "$SCA_DIR/summary.txt"
                    echo "" >> "$SCA_DIR/summary.txt"
                    """
                    
                    // Parse and validate results
                    def scaResults = sh(
                        script: """
                        if [ -f "$SCA_DIR/sca-report.json" ]; then
                            cat "$SCA_DIR/sca-report.json"
                        else
                            echo '[]'
                        fi
                        """,
                        returnStdout: true
                    ).trim()
                    
                    // Calculate new streak for display
                    def currentStreak = env.CLEAN_BUILD_STREAK.toInteger()
                    def newStreakDisplay = currentStreak + 1
                    
                    // Check thresholds
                    sh """
                    # Count violations by severity
                    CRITICAL_COUNT=0
                    HIGH_COUNT=0
                    
                    if [ -f "$SCA_DIR/sca-report.csv" ]; then
                        CRITICAL_COUNT=\$(grep -i "critical" "$SCA_DIR/sca-report.csv" | wc -l || echo 0)
                        HIGH_COUNT=\$(grep -i "high" "$SCA_DIR/sca-report.csv" | wc -l || echo 0)
                    fi
                    
                    echo "Critical Issues: \$CRITICAL_COUNT" >> "$SCA_DIR/summary.txt"
                    echo "High Issues: \$HIGH_COUNT" >> "$SCA_DIR/summary.txt"
                    echo "" >> "$SCA_DIR/summary.txt"
                    
                    # Threshold validation
                    if [ \$CRITICAL_COUNT -gt ${SCA_THRESHOLD_CRITICAL} ]; then
                        echo "❌ FAILED: \$CRITICAL_COUNT Critical issues found (threshold: ${SCA_THRESHOLD_CRITICAL})" >> "$SCA_DIR/summary.txt"
                        echo "⚠️ WARNING: Critical security/quality issues detected!"
                        exit 1
                    fi
                    
                    if [ \$HIGH_COUNT -gt ${SCA_THRESHOLD_HIGH} ]; then
                        echo "⚠️ WARNING: \$HIGH_COUNT High severity issues found (threshold: ${SCA_THRESHOLD_HIGH})" >> "$SCA_DIR/summary.txt"
                        echo "⚠️ WARNING: High severity issues detected!"
                        exit 1
                    fi
                    
                    echo "✅ PASSED: No Critical/High issues above threshold" >> "$SCA_DIR/summary.txt"
                    echo "" >> "$SCA_DIR/summary.txt"
                    echo "Consecutive Clean Builds: ${currentStreak} → ${newStreakDisplay}" >> "$SCA_DIR/summary.txt"
                    cat "$SCA_DIR/summary.txt"
                    """
                }
            }
            post {
                always {
                    // Archive all SCA reports
                    archiveArtifacts artifacts: "${SCA_DIR}/**/*", allowEmptyArchive: true
                    
                    // Display summary in console
                    sh 'cat "$SCA_DIR/summary.txt" || echo "No summary available"'
                }
                failure {
                    echo '❌ Static Code Analysis failed - Critical/High severity issues detected'
                }
            }
        }

        /* ---------------------------------------------------------
           Generate Delta Package (if DELTA scope)
           --------------------------------------------------------- */
        stage('Generate Delta') {
            when {
                expression { params.DEPLOY_SCOPE == 'DELTA' }
            }
            steps {
                script {
                    echo "=== Generating Delta Package ==="
                    echo "Baseline Tag: ${env.BASELINE_TAG}"
                    echo "Current HEAD: ${env.GIT_COMMIT_HASH}"
                    
                    sh """
                    mkdir -p "${DELTA_DIR}"
                    
                    # Generate delta using sfdx-git-delta
                    echo "Comparing ${env.BASELINE_TAG} to HEAD..."
                    
                    "${SF}" sgd:source:delta \
                      --to HEAD \
                      --from "${env.BASELINE_TAG}" \
                      --output-dir "${DELTA_DIR}" \
                      --generate-delta \
                      --source-dir force-app
                    
                    echo ""
                    echo "Delta package generated in ${DELTA_DIR}"
                    echo "Contents:"
                    ls -la "${DELTA_DIR}" || true
                    
                    # Check if there are changes
                    if [ -f "${DELTA_DIR}/package/package.xml" ]; then
                        echo ""
                        echo "Changes detected:"
                        cat "${DELTA_DIR}/package/package.xml"
                    else
                        echo "⚠️  No changes detected between ${env.BASELINE_TAG} and HEAD"
                        echo "Consider using FULL deployment or check if baseline tag is current"
                    fi
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: "${DELTA_DIR}/**/*", allowEmptyArchive: true
                }
            }
        }

        /* ---------------------------------------------------------
           Validate Deployment (Dry Run)
           --------------------------------------------------------- */
        stage('Validate Deployment') {
            steps {
                script {
                    def startTime = System.currentTimeMillis()

                    if (params.APEX_CLASSES.trim()) {
                        def classPaths = params.APEX_CLASSES
                            .split(',')
                            .collect { it.trim() }
                            .collect { "force-app/main/default/classes/${it}.cls" }
                            .join(',')

                        sh """
                        rm -rf .sf .sfdx
                        "$SF" project deploy start \
                          --source-dir ${classPaths} \
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
                        "$SF" project deploy start \
                          --manifest "${DELTA_DIR}/package/package.xml" \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --dry-run \
                          --wait 60
                        """
                        return
                    }

                    if (params.DEPLOY_FORMAT == 'MDAPI') {
                        sh """
                        rm -rf .sf .sfdx
                        "$SF" project deploy start \
                          --manifest ${params.PACKAGE_XML_PATH} \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --dry-run \
                          --wait 60
                        """
                        return
                    }

                    // FULL deployment only
                    sh """
                    rm -rf .sf .sfdx
                    "$SF" project deploy start \
                      --source-dir force-app \
                      --target-org "$SF_ALIAS" \
                      --test-level "$TEST_LEVEL" \
                      --dry-run \
                      --wait 60
                    """
                    
                    def duration = System.currentTimeMillis() - startTime
                    echo "✅ Validation completed in ${duration}ms"
                    
                    // Log validation metrics
                    sh """
                    mkdir -p logs
                    echo "{\"stage\": \"Validate\", \"duration_ms\": ${duration}, \"timestamp\": \"\$(date -Iseconds)\"}" >> "$BUILD_METRICS"
                    """
                }
            }
            post {
                failure {
                    script {
                        sh """
                        mkdir -p logs
                        echo "❌ Validation Failed" > logs/validation-failure.log
                        echo "Build: ${BUILD_NUMBER}" >> logs/validation-failure.log
                        echo "Format: ${params.DEPLOY_FORMAT}" >> logs/validation-failure.log
                        echo "Scope: ${params.DEPLOY_SCOPE}" >> logs/validation-failure.log
                        echo "Timestamp: \$(date)" >> logs/validation-failure.log
                        cat logs/validation-failure.log
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
                    def startTime = System.currentTimeMillis()
                    
                    echo "=== Starting Deployment ==="
                    echo "Build: ${BUILD_NUMBER}"
                    echo "Environment: ${params.TARGET_ENV}"
                    echo "Branch: ${env.GIT_BRANCH}"
                    echo "Format: ${params.DEPLOY_FORMAT}"
                    echo "Scope: ${params.DEPLOY_SCOPE}"
                    echo "Test Level: ${params.TEST_LEVEL}"
                    echo "Timestamp: ${new Date()}"

                    if (params.APEX_CLASSES.trim()) {
                        def classPaths = params.APEX_CLASSES
                            .split(',')
                            .collect { it.trim() }
                            .collect { "force-app/main/default/classes/${it}.cls" }
                            .join(',')

                        sh """
                        rm -rf .sf .sfdx
                        "$SF" project deploy start \
                          --source-dir ${classPaths} \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --wait 60
                        """
                        return
                    }

                    if (params.DEPLOY_SCOPE == 'DELTA') {
                        sh """
                        rm -rf .sf .sfdx
                        "$SF" project deploy start \
                          --manifest "${DELTA_DIR}/package/package.xml" \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --wait 60
                        """
                        
                        def duration = System.currentTimeMillis() - startTime
                        echo "✅ Delta deployment completed in ${duration}ms"
                        return
                    }

                    if (params.DEPLOY_FORMAT == 'MDAPI') {
                        sh """
                        rm -rf .sf .sfdx
                        "$SF" project deploy start \
                          --manifest ${params.PACKAGE_XML_PATH} \
                          --target-org "$SF_ALIAS" \
                          --test-level "$TEST_LEVEL" \
                          --wait 60
                        """
                        return
                    }

                    // FULL deployment only
                    sh """
                    rm -rf .sf .sfdx
                    "$SF" project deploy start \
                      --source-dir force-app \
                      --target-org "$SF_ALIAS" \
                      --test-level "$TEST_LEVEL" \
                      --wait 60
                    """
                    
                    def duration = System.currentTimeMillis() - startTime
                    echo "✅ Deployment completed in ${duration}ms"
                    
                    // Log deployment success metrics
                    sh """
                    mkdir -p logs
                    echo "{\"stage\": \"Deploy\", \"duration_ms\": ${duration}, \"timestamp\": \"\$(date -Iseconds)\", \"status\": \"success\"}" >> "$BUILD_METRICS"
                    
                    # Create deployment summary
                    echo "=== Deployment Summary ===" > logs/deployment-summary.txt
                    echo "Build: ${BUILD_NUMBER}" >> logs/deployment-summary.txt
                    echo "Status: SUCCESS" >> logs/deployment-summary.txt
                    echo "Duration: ${duration}ms" >> logs/deployment-summary.txt
                    echo "Format: ${params.DEPLOY_FORMAT}" >> logs/deployment-summary.txt
                    echo "Scope: ${params.DEPLOY_SCOPE}" >> logs/deployment-summary.txt
                    echo "Test Level: ${params.TEST_LEVEL}" >> logs/deployment-summary.txt
                    echo "Timestamp: \$(date)" >> logs/deployment-summary.txt
                    echo "Git Commit: ${env.GIT_COMMIT_HASH}" >> logs/deployment-summary.txt
                    echo "" >> logs/deployment-summary.txt
                    echo "Rollback Instructions:" >> logs/deployment-summary.txt
                    echo "1. Click 'Build with Parameters'" >> logs/deployment-summary.txt
                    echo "2. Check 'ROLLBACK_MODE'" >> logs/deployment-summary.txt
                    echo "3. Select this build (#${BUILD_NUMBER}) or any previous build" >> logs/deployment-summary.txt
                    echo "4. Click 'Build' to initiate rollback" >> logs/deployment-summary.txt
                    
                    cat logs/deployment-summary.txt
                    """
                }
            }
            post {
                success {
                    archiveArtifacts artifacts: 'logs/deployment-summary.txt', allowEmptyArchive: true
                }
                failure {
                    script {
                        sh """
                        mkdir -p logs
                        echo "=== Deployment Failure Details ===" > logs/deployment-failure.log
                        echo "Build: ${BUILD_NUMBER}" >> logs/deployment-failure.log
                        echo "Failed Stage: Deploy" >> logs/deployment-failure.log
                        echo "Format: ${params.DEPLOY_FORMAT}" >> logs/deployment-failure.log
                        echo "Scope: ${params.DEPLOY_SCOPE}" >> logs/deployment-failure.log
                        echo "Timestamp: \$(date)" >> logs/deployment-failure.log
                        echo "" >> logs/deployment-failure.log
                        echo "Recommended Actions:" >> logs/deployment-failure.log
                        echo "1. Review deployment logs above" >> logs/deployment-failure.log
                        echo "2. Check for test failures or metadata conflicts" >> logs/deployment-failure.log
                        echo "3. Verify org connectivity and permissions" >> logs/deployment-failure.log
                        echo "4. Consider rollback if partial deployment occurred" >> logs/deployment-failure.log
                        echo "" >> logs/deployment-failure.log
                        echo "Rollback Command (if needed):" >> logs/deployment-failure.log
                        echo "git revert HEAD && re-run pipeline" >> logs/deployment-failure.log
                        
                        cat logs/deployment-failure.log
                        """
                        
                        archiveArtifacts artifacts: 'logs/deployment-failure.log', allowEmptyArchive: true
                    }
                }
            }
        }
        
        /* ---------------------------------------------------------
           Post-Deployment Validation
           --------------------------------------------------------- */
        stage('Post-Deployment Validation') {
            steps {
                script {
                    echo "=== Running Post-Deployment Checks ==="
                    
                    sh """
                    # Verify org connectivity
                    "$SF" org display --target-org "$SF_ALIAS" --json
                    
                    # Generate final build metrics
                    mkdir -p logs
                    echo "=== Build Metrics ===" > logs/build-metrics-summary.txt
                    echo "Build Number: ${BUILD_NUMBER}" >> logs/build-metrics-summary.txt
                    echo "Build Date: \$(date)" >> logs/build-metrics-summary.txt
                    echo "Format: ${params.DEPLOY_FORMAT}" >> logs/build-metrics-summary.txt
                    echo "Scope: ${params.DEPLOY_SCOPE}" >> logs/build-metrics-summary.txt
                    echo "" >> logs/build-metrics-summary.txt
                    
                    if [ -f "$BUILD_METRICS" ]; then
                        echo "Stage Timings:" >> logs/build-metrics-summary.txt
                        cat "$BUILD_METRICS" >> logs/build-metrics-summary.txt
                    fi
                    
                    cat logs/build-metrics-summary.txt
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'logs/**/*,*.json,', allowEmptyArchive: true
                }
            }
        }

        /* ---------------------------------------------------------
           Update Baseline Tag (on successful deployment)
           --------------------------------------------------------- */
        stage('Update Baseline Tag') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            steps {
                script {
                    echo "=== Updating Baseline Tag ==="
                    echo "Tag: ${env.BASELINE_TAG}"
                    echo "Commit: ${env.GIT_COMMIT_HASH}"
                    
                    withCredentials([usernamePassword(
                        credentialsId: 'supercred',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )]) {
                        sh '''
                        # Configure Git
                        git config user.email "jenkins@ci.com"
                        git config user.name "Jenkins CI"
                        
                        # Delete existing tag locally and remotely
                        git tag -d "$BASELINE_TAG" || true
                        git push https://"$GIT_USERNAME":"$GIT_PASSWORD"@github.com/manishamondal/salesforce-full.git :refs/tags/"$BASELINE_TAG" || true
                        
                        # Create new tag at current commit
                        git tag -a "$BASELINE_TAG" -m "Updated by Jenkins Build #${BUILD_NUMBER} on $(date)"
                        
                        # Push tag to remote
                        git push https://"$GIT_USERNAME":"$GIT_PASSWORD"@github.com/manishamondal/salesforce-full.git "$BASELINE_TAG"
                        
                        echo "✅ Baseline tag $BASELINE_TAG updated to ${GIT_COMMIT_HASH}"
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                // Generate comprehensive build report
                sh """
                mkdir -p logs
                
                echo "========================================" > logs/pipeline-summary.txt
                echo "   SALESFORCE CI/CD PIPELINE SUMMARY   " >> logs/pipeline-summary.txt
                echo "========================================" >> logs/pipeline-summary.txt
                echo "" >> logs/pipeline-summary.txt
                echo "Build Number: ${BUILD_NUMBER}" >> logs/pipeline-summary.txt
                echo "Build URL: ${BUILD_URL}" >> logs/pipeline-summary.txt
                echo "Status: ${currentBuild.result ?: 'IN_PROGRESS'}" >> logs/pipeline-summary.txt
                echo "Duration: ${currentBuild.durationString}" >> logs/pipeline-summary.txt
                echo "Started: ${new Date(currentBuild.startTimeInMillis)}" >> logs/pipeline-summary.txt
                echo "" >> logs/pipeline-summary.txt
                echo "Configuration:" >> logs/pipeline-summary.txt
                echo "  - Deploy Format: ${params.DEPLOY_FORMAT}" >> logs/pipeline-summary.txt
                echo "  - Deploy Scope: ${params.DEPLOY_SCOPE}" >> logs/pipeline-summary.txt
                echo "  - Test Level: ${params.TEST_LEVEL}" >> logs/pipeline-summary.txt
                echo "  - Apex Classes: ${params.APEX_CLASSES ?: 'All'}" >> logs/pipeline-summary.txt
                echo "  - Target Org Alias: ${SF_ALIAS}" >> logs/pipeline-summary.txt

                echo "" >> logs/pipeline-summary.txt
                
                # Include SCA summary if available
                if [ -f "${SCA_DIR}/summary.txt" ]; then
                    echo "Static Code Analysis:" >> logs/pipeline-summary.txt
                    cat "${SCA_DIR}/summary.txt" >> logs/pipeline-summary.txt
                    echo "" >> logs/pipeline-summary.txt
                fi
                
                cat logs/pipeline-summary.txt
                """
                
                // Archive all reports and logs
                archiveArtifacts artifacts: 'logs/**/*', allowEmptyArchive: true
            }
        }
        
        success {
            script {
                // Calculate new streak
                def newStreak = (env.CLEAN_BUILD_STREAK.toInteger() + 1)
                
                echo ''
                echo '✅ ========================================'
                echo '✅  SALESFORCE DEPLOYMENT SUCCESSFUL'
                echo '✅ ========================================'
                echo ''
                echo "Build: ${BUILD_NUMBER}"
                echo "Duration: ${currentBuild.durationString}"
                echo "SCA Report: ${BUILD_URL}artifact/${SCA_DIR}/sca-report.html"
                echo "Build Logs: ${BUILD_URL}console"
                echo ''
                echo '🏆 Code Quality Streak'
                echo "   Consecutive Clean Builds: ${newStreak}"
                if (newStreak >= 2) {
                    echo "   ✅ TARGET ACHIEVED: 2+ consecutive deployments with 0 Critical/High issues!"
                } else {
                    echo "   📊 Continue to reach target of 2 consecutive clean builds"
                }
                echo ''
                
                sh """
                mkdir -p logs
                echo "success" > logs/build-status.txt
                echo "Build ${BUILD_NUMBER} completed successfully at \$(date)" >> logs/build-status.txt
                echo "" >> logs/build-status.txt
                
                # Store git commit hash for rollback capability
                echo "${env.GIT_COMMIT_HASH}" > logs/git-commit.txt
                
                # Store streak for next build
                echo "${newStreak}" > logs/sca-streak.txt
                
                # Store detailed streak info
                echo "=== Code Quality Streak ==="  > logs/sca-streak-details.txt
                echo "Build Number: ${BUILD_NUMBER}" >> logs/sca-streak-details.txt
                echo "Date: \$(date)" >> logs/sca-streak-details.txt
                echo "Consecutive Clean Builds: ${newStreak}" >> logs/sca-streak-details.txt
                echo "Critical Issues: 0" >> logs/sca-streak-details.txt
                echo "High Issues: 0" >> logs/sca-streak-details.txt
                echo "" >> logs/sca-streak-details.txt
                
                if [ ${newStreak} -ge 2 ]; then
                    echo "✅ OBJECTIVE ACHIEVED: 2+ consecutive clean deployments" >> logs/sca-streak-details.txt
                    echo "This demonstrates:" >> logs/sca-streak-details.txt
                    echo "- No new Critical/High issues introduced" >> logs/sca-streak-details.txt
                    echo "- Consistent code quality standards" >> logs/sca-streak-details.txt
                    echo "- Security & quality awareness embedded in CI/CD" >> logs/sca-streak-details.txt
                else
                    echo "📊 Progress: ${newStreak} of 2 consecutive clean builds" >> logs/sca-streak-details.txt
                fi
                
                cat logs/sca-streak-details.txt
                """
            }
        }
        
        failure {
            script {
                echo ''
                echo '❌ ========================================'
                echo '❌  SALESFORCE PIPELINE FAILED'
                echo '❌ ========================================'
                echo ''
                echo "Build: ${BUILD_NUMBER}"
                echo "Failed Stage: Check console output above"
                echo "Duration: ${currentBuild.durationString}"
                echo ''
                echo 'Troubleshooting Steps:'
                echo '1. Review console logs for specific error messages'
                echo '2. Check SCA report for code quality issues'
                echo '3. Verify Salesforce org connectivity and permissions'
                echo '4. Review deployment validation errors'
                echo '5. Check if tests failed (if TEST_LEVEL=RunLocalTests)'
                echo ''
                echo "Console Logs: ${BUILD_URL}console"
                echo "Artifacts: ${BUILD_URL}artifact/"
                echo ''
                
                sh """
                mkdir -p logs
                echo "failure" > logs/build-status.txt
                echo "Build ${BUILD_NUMBER} failed at \$(date)" >> logs/build-status.txt
                echo "" >> logs/build-status.txt
                
                # Reset streak to 0 on failure
                echo "0" > logs/sca-streak.txt
                echo "⚠️ Consecutive clean build streak reset to 0" >> logs/build-status.txt
                echo "Previous streak: ${CLEAN_BUILD_STREAK}" >> logs/build-status.txt
                echo "" >> logs/build-status.txt
                
                echo "Common Failure Categories:" >> logs/build-status.txt
                echo "1. CODE_QUALITY: SCA found Critical/High issues" >> logs/build-status.txt
                echo "2. VALIDATION: Metadata validation failed" >> logs/build-status.txt
                echo "3. TEST_FAILURE: Apex tests failed" >> logs/build-status.txt
                echo "4. AUTHENTICATION: SF org connection issues" >> logs/build-status.txt
                echo "5. CONFIGURATION: Pipeline parameter issues" >> logs/build-status.txt
                echo "" >> logs/build-status.txt
                echo "Next Steps:" >> logs/build-status.txt
                echo "- Document this failure in RCA template" >> logs/build-status.txt
                echo "- Identify if this is a recurring issue" >> logs/build-status.txt
                echo "- Implement preventive measures" >> logs/build-status.txt
                echo "- Fix issues to restart clean build streak" >> logs/build-status.txt
                """
            }
        }
        
        aborted {
            script {
                echo ''
                echo '⚠️  ========================================'
                echo '⚠️   DEPLOYMENT ABORTED'
                echo '⚠️  ========================================'
                echo ''
                echo "Build: ${BUILD_NUMBER}"
                echo "Reason: User abort or approval timeout"
                echo "Duration: ${currentBuild.durationString}"
                echo ''
                
                sh """
                mkdir -p logs
                echo "aborted" > logs/build-status.txt
                echo "Build ${BUILD_NUMBER} was aborted at \$(date)" >> logs/build-status.txt
                """
            }
        }
    }
}
