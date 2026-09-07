// =============================================================================
// TASK 7.1C
// DEVSECOPS PIPELINE WITH EXTENDED EMAIL NOTIFICATIONS
// Platform: Windows Jenkins
// Repository: https://github.com/auma21/8.2CDevSecOps.git
//
// Part 1 - Task 2:
//   - Checkout nodejs-goof
//   - Install dependencies
//   - Run tests
//   - Generate coverage report
//   - Run npm audit security scan
//
// Part 2 - Task 2:
//   - Send an email after Run Tests
//   - Send an email after NPM Audit
//   - Report SUCCESS or FAILURE
//   - Attach stage-specific logs
//   - Attach Jenkins console log
//
// =============================================================================

pipeline {

    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    triggers {
        // The assessment permits SCM polling instead of a webhook.
        pollSCM('H/2 * * * *')
    }

    environment {
        REPO_URL = 'https://github.com/auma21/8.2CDevSecOps.git'
        EMAIL_TO = 'aumarbles@gmail.com'
    }

    stages {

        // =====================================================================
        // STAGE 1: CHECKOUT
        // Tool: Git / GitHub
        // =====================================================================
        stage('Checkout') {
            steps {
                echo '=========================================================='
                echo 'STAGE 1: CHECKOUT'
                echo 'Tool: Git / GitHub'
                echo '=========================================================='

                // Prevent old logs from being reused in a new build.
                deleteDir()

                echo "Checking out repository: ${env.REPO_URL}"

                git(
                    branch: 'main',
                    url: "${env.REPO_URL}"
                )

                echo 'Repository checkout completed.'
            }
        }

        // =====================================================================
        // STAGE 2: INSTALL DEPENDENCIES
        // Tool: npm
        // =====================================================================
        stage('Install Dependencies') {
            steps {
                echo '=========================================================='
                echo 'STAGE 2: INSTALL DEPENDENCIES'
                echo 'Tool: npm'
                echo '=========================================================='

                bat 'node --version'
                bat 'npm --version'

                echo 'Installing Node.js dependencies...'
                bat 'npm install'

                echo 'Dependency installation completed.'
            }
        }

        // =====================================================================
        // STAGE 3: RUN TESTS
        // Tool: npm / project-configured test command
        // =====================================================================
        stage('Run Tests') {

            steps {
                script {
                    echo '=========================================================='
                    echo 'STAGE 3: RUN TESTS'
                    echo 'Tool: npm / project-configured test command'
                    echo '=========================================================='

                    // Runtime status variable. Do not declare this in the
                    // top-level environment block.
                    env.TEST_STAGE_STATUS = 'NOT_RUN'

                    bat 'if not exist logs mkdir logs'

                    echo 'Running npm test...'

                    int testExitCode = bat(
                        returnStatus: true,
                        script: '''
                            @echo off
                            npm test > logs\\npm-test.log 2>&1
                        '''
                    )

                    echo "npm test exit code = ${testExitCode}"

                    if (testExitCode == 0) {
                        env.TEST_STAGE_STATUS = 'SUCCESS'
                        echo 'Run Tests completed successfully.'
                    } else {
                        env.TEST_STAGE_STATUS = 'FAILURE'
                        echo "Run Tests returned exit code ${testExitCode}."

                        catchError(
                            buildResult: 'SUCCESS',
                            stageResult: 'FAILURE'
                        ) {
                            error(
                                "Run Tests failed with exit code ${testExitCode}."
                            )
                        }
                    }

                    echo '---------------- TEST LOG ----------------'
                    bat 'type logs\\npm-test.log'
                    echo '-------------- END TEST LOG --------------'

                    echo "Final Run Tests status = ${env.TEST_STAGE_STATUS}"
                }
            }

            // =================================================================
            // TEST-STAGE EMAIL NOTIFICATION
            // =================================================================
            post {
                always {
                    script {
                        def testStatus = env.TEST_STAGE_STATUS ?: 'FAILURE'

                        echo '=========================================================='
                        echo 'PREPARING RUN TESTS EMAIL'
                        echo "TEST_STAGE_STATUS before email = ${testStatus}"
                        echo '=========================================================='

                        emailext(
                            to: "${env.EMAIL_TO}",

                            subject:
                                "[Jenkins] Run Tests ${testStatus} - " +
                                "${env.JOB_NAME} #${env.BUILD_NUMBER}",

                            mimeType: 'text/html',

                            body: """
                                <html>
                                <body>

                                    <h2>Jenkins Test Stage Notification</h2>

                                    <table border="1" cellpadding="6" cellspacing="0">
                                        <tr>
                                            <td><b>Job</b></td>
                                            <td>${env.JOB_NAME}</td>
                                        </tr>
                                        <tr>
                                            <td><b>Build Number</b></td>
                                            <td>${env.BUILD_NUMBER}</td>
                                        </tr>
                                        <tr>
                                            <td><b>Stage</b></td>
                                            <td>Run Tests</td>
                                        </tr>
                                        <tr>
                                            <td><b>Status</b></td>
                                            <td>${testStatus}</td>
                                        </tr>
                                        <tr>
                                            <td><b>Build URL</b></td>
                                            <td>
                                                <a href="${env.BUILD_URL}">${env.BUILD_URL}</a>
                                            </td>
                                        </tr>
                                    </table>

                                    <p>
                                        The Run Tests stage has completed.
                                        The stage-specific npm test log and Jenkins console log are attached.
                                    </p>

                                    <p>
                                        <b>Note:</b>
                                        FAILURE means the project's configured npm test command returned
                                        a non-zero exit code. The pipeline continues so the remaining
                                        DevSecOps stages can still execute.
                                    </p>

                                </body>
                                </html>
                            """,

                            attachLog: true,
                            compressLog: true,
                            attachmentsPattern: 'logs/npm-test.log'
                        )

                        echo 'Run Tests notification email processed.'
                    }
                }
            }
        }

        // =====================================================================
        // STAGE 4: GENERATE COVERAGE REPORT
        // Tool: npm
        // =====================================================================
        stage('Generate Coverage Report') {

            steps {
                script {
                    echo '=========================================================='
                    echo 'STAGE 4: GENERATE COVERAGE REPORT'
                    echo 'Tool: npm'
                    echo '=========================================================='

                    env.COVERAGE_STAGE_STATUS = 'NOT_RUN'

                    bat 'if not exist logs mkdir logs'

                    echo 'Executing npm run coverage...'

                    int coverageExitCode = bat(
                        returnStatus: true,
                        script: '''
                            @echo off
                            npm run coverage > logs\\coverage.log 2>&1
                        '''
                    )

                    echo "Coverage command exit code = ${coverageExitCode}"

                    if (coverageExitCode == 0) {
                        env.COVERAGE_STAGE_STATUS = 'SUCCESS'
                        echo 'Coverage command completed successfully.'
                    } else {
                        env.COVERAGE_STAGE_STATUS = 'FAILURE'
                        echo "Coverage command returned exit code ${coverageExitCode}."
                        echo 'Pipeline execution will continue as required by the assessment.'
                    }

                    echo '--------------- COVERAGE LOG ---------------'
                    bat 'type logs\\coverage.log'
                    echo '------------- END COVERAGE LOG -------------'

                    echo "Final Coverage status = ${env.COVERAGE_STAGE_STATUS}"
                }
            }
        }

        // =====================================================================
        // STAGE 5: NPM AUDIT SECURITY SCAN
        // Tool: npm audit
        // =====================================================================
        stage('NPM Audit (Security Scan)') {

            steps {
                script {
                    echo '=========================================================='
                    echo 'STAGE 5: NPM AUDIT SECURITY SCAN'
                    echo 'Tool: npm audit'
                    echo '=========================================================='

                    env.SECURITY_STAGE_STATUS = 'NOT_RUN'

                    bat 'if not exist logs mkdir logs'

                    echo 'Running npm audit...'

                    int auditExitCode = bat(
                        returnStatus: true,
                        script: '''
                            @echo off
                            npm audit > logs\\npm-audit.log 2>&1
                        '''
                    )

                    echo "npm audit exit code = ${auditExitCode}"

                    if (auditExitCode == 0) {
                        env.SECURITY_STAGE_STATUS = 'SUCCESS'
                        echo 'NPM Audit completed without a failing result.'
                    } else {
                        env.SECURITY_STAGE_STATUS = 'FAILURE'
                        echo "NPM Audit returned exit code ${auditExitCode}."

                        catchError(
                            buildResult: 'SUCCESS',
                            stageResult: 'FAILURE'
                        ) {
                            error(
                                "NPM Audit identified security issues. Exit code: ${auditExitCode}."
                            )
                        }
                    }

                    echo '--------------- NPM AUDIT LOG ---------------'
                    bat 'type logs\\npm-audit.log'
                    echo '------------- END NPM AUDIT LOG -------------'

                    echo "Final NPM Audit status = ${env.SECURITY_STAGE_STATUS}"
                }
            }

            // =================================================================
            // SECURITY-SCAN EMAIL NOTIFICATION
            // =================================================================
            post {
                always {
                    script {
                        def securityStatus = env.SECURITY_STAGE_STATUS ?: 'FAILURE'

                        echo '=========================================================='
                        echo 'PREPARING NPM AUDIT EMAIL'
                        echo "SECURITY_STAGE_STATUS before email = ${securityStatus}"
                        echo '=========================================================='

                        emailext(
                            to: "${env.EMAIL_TO}",

                            subject:
                                "[Jenkins] Security Scan ${securityStatus} - " +
                                "${env.JOB_NAME} #${env.BUILD_NUMBER}",

                            mimeType: 'text/html',

                            body: """
                                <html>
                                <body>

                                    <h2>Jenkins Security Scan Notification</h2>

                                    <table border="1" cellpadding="6" cellspacing="0">
                                        <tr>
                                            <td><b>Job</b></td>
                                            <td>${env.JOB_NAME}</td>
                                        </tr>
                                        <tr>
                                            <td><b>Build Number</b></td>
                                            <td>${env.BUILD_NUMBER}</td>
                                        </tr>
                                        <tr>
                                            <td><b>Stage</b></td>
                                            <td>NPM Audit (Security Scan)</td>
                                        </tr>
                                        <tr>
                                            <td><b>Status</b></td>
                                            <td>${securityStatus}</td>
                                        </tr>
                                        <tr>
                                            <td><b>Build URL</b></td>
                                            <td>
                                                <a href="${env.BUILD_URL}">${env.BUILD_URL}</a>
                                            </td>
                                        </tr>
                                    </table>

                                    <p>
                                        The npm audit security scan has completed.
                                        The stage-specific npm audit log and Jenkins console log are attached.
                                    </p>

                                    <p>
                                        <b>Interpretation:</b>
                                        FAILURE can indicate that npm audit successfully detected known
                                        vulnerabilities in the intentionally vulnerable nodejs-goof application.
                                    </p>

                                </body>
                                </html>
                            """,

                            attachLog: true,
                            compressLog: true,
                            attachmentsPattern: 'logs/npm-audit.log'
                        )

                        echo 'Security Scan notification email processed.'
                    }
                }
            }
        }
    }

    // =========================================================================
    // PIPELINE-LEVEL POST PROCESSING
    // =========================================================================
    post {

        always {
            script {
                echo '=========================================================='
                echo 'PIPELINE POST-PROCESSING'
                echo '=========================================================='

                archiveArtifacts(
                    artifacts: 'logs/*.log',
                    allowEmptyArchive: true
                )

                echo 'Generated logs archived as Jenkins build artefacts.'

                def finalTestStatus = env.TEST_STAGE_STATUS ?: 'NOT_RUN'
                def finalCoverageStatus = env.COVERAGE_STAGE_STATUS ?: 'NOT_RUN'
                def finalSecurityStatus = env.SECURITY_STAGE_STATUS ?: 'NOT_RUN'

                echo "Run Tests status: ${finalTestStatus}"
                echo "Coverage status: ${finalCoverageStatus}"
                echo "Security Scan status: ${finalSecurityStatus}"

                echo 'Task 7.1C DevSecOps Pipeline execution completed.'
            }
        }
    }
}
