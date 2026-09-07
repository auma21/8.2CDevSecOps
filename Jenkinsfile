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
// =============================================================================

pipeline {

    // =========================================================================
    // AGENT
    // =========================================================================
    agent any


    // =========================================================================
    // PIPELINE OPTIONS
    // =========================================================================
    options {

        /*
         * Jenkins normally performs an automatic SCM checkout when the
         * Jenkinsfile is loaded from source control.
         *
         */
        skipDefaultCheckout(true)

        /*
         * Add timestamps to the Jenkins Console Output.
         */
        timestamps()
    }


    // =========================================================================
    // AUTOMATIC GITHUB POLLING
    // =========================================================================
    triggers {

        /*
         * Jenkins checks the GitHub repository approximately every two minutes
         * for a new commit.
         *
         * Allows SCM polling instead of a GitHub webhook.
         */
        pollSCM('H/2 * * * *')
    }


    // =========================================================================
    // GLOBAL ENVIRONMENT VARIABLES
    // =========================================================================
    environment {

        /*
         * GitHub repository containing the nodejs-goof project.
         */
        REPO_URL = 'https://github.com/auma21/8.2CDevSecOps.git'

        /*
         * Email address that should receive Jenkins notifications.
         */
        EMAIL_TO = 'aumarbles@gmail.com'

        /*
         * Initial values.
         *
         * These values are replaced with SUCCESS or FAILURE when the
         * corresponding commands execute.
         */
        TEST_STAGE_STATUS = 'NOT_RUN'

        SECURITY_STAGE_STATUS = 'NOT_RUN'

        COVERAGE_STAGE_STATUS = 'NOT_RUN'
    }


    // =========================================================================
    // PIPELINE STAGES
    // =========================================================================
    stages {


        // =====================================================================
        // STAGE 1: CHECKOUT
        //
        // Task:
        // Retrieve the nodejs-goof project from GitHub.
        //
        // Tool:
        // Git / GitHub
        // =====================================================================
        stage('Checkout') {

            steps {

                echo '=========================================================='
                echo 'STAGE 1: CHECKOUT'
                echo 'Tool: Git / GitHub'
                echo '=========================================================='

                /*
                 * Remove files left over from previous builds.
                 *
                 * This is particularly important for the logs directory.
                 * Otherwise an old log could accidentally be attached to a
                 * new email.
                 */
                deleteDir()

                echo "Checking out repository:"
                echo "${env.REPO_URL}"

                git(
                    branch: 'main',
                    url: "${env.REPO_URL}"
                )

                echo 'Repository checkout completed.'
            }
        }


        // =====================================================================
        // STAGE 2: INSTALL DEPENDENCIES
        //
        // Task:
        // Install dependencies defined in package.json.
        //
        // Tool:
        // npm
        // =====================================================================
        stage('Install Dependencies') {

            steps {

                echo '=========================================================='
                echo 'STAGE 2: INSTALL DEPENDENCIES'
                echo 'Tool: npm'
                echo '=========================================================='

                /*
                 * Verifying that Jenkins can locate Node.js and npm.
                 */
                bat 'node --version'
                bat 'npm --version'

                echo 'Installing Node.js dependencies...'
                bat 'npm install'

                echo 'Dependency installation completed.'
            }
        }


        // =====================================================================
        // STAGE 3: RUN TESTS
        //
        // Task:
        // Execute the test command configured by nodejs-goof.
        //
        // Tool:
        // npm / project-configured test tool
        //
        // Important:
        // The current nodejs-goof npm test command invokes Snyk.
        // Without Snyk authentication this can return a non-zero exit code.
        //
        // =====================================================================
        stage('Run Tests') {

            steps {

                script {

                    echo '=========================================================='
                    echo 'STAGE 3: RUN TESTS'
                    echo 'Tool: npm / project-configured test command'
                    echo '=========================================================='

                    /*
                     * Create a folder for stage-specific logs.
                     */
                    bat 'if not exist logs mkdir logs'

                    echo 'Running npm test...'

                    /*
                     * Execute npm test and save all output in npm-test.log.
                     *
                     * returnStatus: true prevents a non-zero exit code from
                     * immediately terminating the whole pipeline.
                     *
                     * The actual return code is stored so the notification
                     * can accurately report SUCCESS or FAILURE.
                     */
                    int testExitCode = bat(
                        returnStatus: true,
                        script: '''
                            @echo off
                            npm test > logs\\npm-test.log 2>&1
                        '''
                    )

                    echo "npm test exit code = ${testExitCode}"

                    /*
                     * Determine the real test result.
                     */
                    if (testExitCode == 0) {

                        env.TEST_STAGE_STATUS = 'SUCCESS'

                        echo 'Run Tests completed successfully.'

                    } else {

                        env.TEST_STAGE_STATUS = 'FAILURE'

                        echo "Run Tests returned exit code ${testExitCode}."

                        /*
                         * Mark this Jenkins stage as failed while allowing
                         * subsequent stages to continue.
                         *
                         * buildResult: SUCCESS
                         *      The overall demonstration pipeline continues.
                         *
                         * stageResult: FAILURE
                         *      Jenkins visually identifies this stage as
                         *      unsuccessful.
                         */
                        catchError(
                            buildResult: 'SUCCESS',
                            stageResult: 'FAILURE'
                        ) {

                            error(
                                "Run Tests failed with exit code ${testExitCode}."
                            )
                        }
                    }

                    /*
                     * Display the saved test output in Jenkins Console Output.
                     */
                    echo '---------------- TEST LOG ----------------'

                    bat 'type logs\\npm-test.log'

                    echo '-------------- END TEST LOG --------------'

                    /*
                     * Diagnostic message confirming the value that will be
                     * used by the stage post block.
                     */
                    echo "Final Run Tests status = ${env.TEST_STAGE_STATUS}"
                }
            }


            // =================================================================
            // TEST-STAGE EMAIL NOTIFICATION
            //
            // IMPORTANT:
            // This post block is INSIDE the Run Tests stage.
            //
            // Therefore it executes only after Run Tests has finished and
            // TEST_STAGE_STATUS has been changed from NOT_RUN.
            // =================================================================
            post {

                always {

                    script {

                        echo '=========================================================='
                        echo 'PREPARING RUN TESTS EMAIL'
                        echo "TEST_STAGE_STATUS before email = ${env.TEST_STAGE_STATUS}"
                        echo '=========================================================='

                        emailext(

                            /*
                             * Recipient.
                             */
                            to: "${env.EMAIL_TO}",


                            /*
                             * Email subject contains:
                             * - stage
                             * - result
                             * - Jenkins job
                             * - build number
                             */
                            subject:
                                "[Jenkins] Run Tests " +
                                "${env.TEST_STAGE_STATUS} - " +
                                "${env.JOB_NAME} " +
                                "#${env.BUILD_NUMBER}",


                            /*
                             * HTML formatted message.
                             */
                            mimeType: 'text/html',


                            /*
                             * Custom email body.
                             */
                            body: """
                                <html>
                                <body>

                                    <h2>
                                        Jenkins Test Stage Notification
                                    </h2>

                                    <table border="1"
                                           cellpadding="6"
                                           cellspacing="0">

                                        <tr>
                                            <td>
                                                <b>Job</b>
                                            </td>

                                            <td>
                                                ${env.JOB_NAME}
                                            </td>
                                        </tr>

                                        <tr>
                                            <td>
                                                <b>Build Number</b>
                                            </td>

                                            <td>
                                                ${env.BUILD_NUMBER}
                                            </td>
                                        </tr>

                                        <tr>
                                            <td>
                                                <b>Stage</b>
                                            </td>

                                            <td>
                                                Run Tests
                                            </td>
                                        </tr>

                                        <tr>
                                            <td>
                                                <b>Status</b>
                                            </td>

                                            <td>
                                                ${env.TEST_STAGE_STATUS}
                                            </td>
                                        </tr>

                                        <tr>
                                            <td>
                                                <b>Build URL</b>
                                            </td>

                                            <td>
                                                <a href="${env.BUILD_URL}">
                                                    ${env.BUILD_URL}
                                                </a>
                                            </td>
                                        </tr>

                                    </table>

                                    <p>
                                        The Run Tests stage has completed.
                                    </p>

                                    <p>
                                        The stage-specific npm test log and
                                        the Jenkins console log are attached
                                        for review.
                                    </p>

                                    <p>
                                        <b>Note:</b>
                                        A FAILURE status indicates that the
                                        project's configured npm test command
                                        returned a non-zero exit code.
                                        The pipeline is configured to continue
                                        so that the remaining DevSecOps stages
                                        can still execute.
                                    </p>

                                </body>
                                </html>
                            """,


                            /*
                             * Attach the complete Jenkins Console Output.
                             */
                            attachLog: true,


                            /*
                             * Compress the Jenkins Console Output attachment
                             * to reduce email size.
                             */
                            compressLog: true,


                            /*
                             * Attach the dedicated test-stage log.
                             */
                            attachmentsPattern:
                                'logs/npm-test.log'
                        )

                        echo 'Run Tests notification email processed.'
                    }
                }
            }
        }


        // =====================================================================
        // STAGE 4: GENERATE COVERAGE REPORT
        //
        // Task:
        // Execute the coverage command requested in the assessment.
        //
        // Tool:
        // npm
        //
        // The pipeline continues if the command does not exist or fails.
        // =====================================================================
        stage('Generate Coverage Report') {

            steps {

                script {

                    echo '=========================================================='
                    echo 'STAGE 4: GENERATE COVERAGE REPORT'
                    echo 'Tool: npm'
                    echo '=========================================================='

                    bat 'if not exist logs mkdir logs'

                    echo 'Executing npm run coverage...'

                    /*
                     * Capture the actual return code without stopping the
                     * overall pipeline.
                     */
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

                        echo """
                            Coverage command returned exit code
                            ${coverageExitCode}.
                        """

                        echo """
                            Pipeline execution will continue as required by
                            the assessment.
                        """
                    }

                    echo '--------------- COVERAGE LOG ---------------'

                    bat 'type logs\\coverage.log'

                    echo '------------- END COVERAGE LOG -------------'

                    echo """
                        Final Coverage status =
                        ${env.COVERAGE_STAGE_STATUS}
                    """
                }
            }
        }


        // =====================================================================
        // STAGE 5: NPM AUDIT SECURITY SCAN
        //
        // Task:
        // Analyse Node.js dependencies for known vulnerabilities.
        //
        // Tool:
        // npm audit
        //
        // nodejs-goof is intentionally vulnerable, so npm audit may
        // legitimately return a non-zero exit code.
        // =====================================================================
        stage('NPM Audit (Security Scan)') {

            steps {

                script {

                    echo '=========================================================='
                    echo 'STAGE 5: NPM AUDIT SECURITY SCAN'
                    echo 'Tool: npm audit'
                    echo '=========================================================='

                    bat 'if not exist logs mkdir logs'

                    echo 'Running npm audit...'

                    /*
                     * Run the security scan and save its results in a
                     * dedicated audit log.
                     */
                    int auditExitCode = bat(
                        returnStatus: true,
                        script: '''
                            @echo off
                            npm audit > logs\\npm-audit.log 2>&1
                        '''
                    )

                    echo "npm audit exit code = ${auditExitCode}"

                    /*
                     * Interpret the audit result.
                     *
                     * npm audit usually returns a non-zero value when
                     * vulnerabilities meeting the configured severity
                     * threshold are detected.
                     */
                    if (auditExitCode == 0) {

                        env.SECURITY_STAGE_STATUS = 'SUCCESS'

                        echo 'NPM Audit completed without a failing result.'

                    } else {

                        env.SECURITY_STAGE_STATUS = 'FAILURE'

                        echo """
                            NPM Audit returned exit code
                            ${auditExitCode}.
                        """

                        /*
                         * Mark this particular stage as failed while keeping
                         * the overall Pipeline available for final
                         * post-processing.
                         */
                        catchError(
                            buildResult: 'SUCCESS',
                            stageResult: 'FAILURE'
                        ) {

                            error(
                                "NPM Audit identified security issues. " +
                                "Exit code: ${auditExitCode}."
                            )
                        }
                    }

                    /*
                     * Display the vulnerability report in Jenkins.
                     */
                    echo '--------------- NPM AUDIT LOG ---------------'

                    bat 'type logs\\npm-audit.log'

                    echo '------------- END NPM AUDIT LOG -------------'

                    /*
                     * Verify the value before the stage's email post block.
                     */
                    echo """
                        Final NPM Audit status =
                        ${env.SECURITY_STAGE_STATUS}
                    """
                }
            }


            // =================================================================
            // SECURITY-SCAN EMAIL NOTIFICATION
            //
            // IMPORTANT:
            // This block is INSIDE the NPM Audit stage and therefore executes
            // only after SECURITY_STAGE_STATUS has been determined.
            // =================================================================
            post {

                always {

                    script {

                        echo '=========================================================='
                        echo 'PREPARING NPM AUDIT EMAIL'
                        echo """
                            SECURITY_STAGE_STATUS before email =
                            ${env.SECURITY_STAGE_STATUS}
                        """
                        echo '=========================================================='

                        emailext(

                            to: "${env.EMAIL_TO}",


                            subject:
                                "[Jenkins] Security Scan " +
                                "${env.SECURITY_STAGE_STATUS} - " +
                                "${env.JOB_NAME} " +
                                "#${env.BUILD_NUMBER}",


                            mimeType: 'text/html',


                            body: """
                                <html>
                                <body>

                                    <h2>
                                        Jenkins Security Scan Notification
                                    </h2>

                                    <table border="1"
                                           cellpadding="6"
                                           cellspacing="0">

                                        <tr>
                                            <td>
                                                <b>Job</b>
                                            </td>

                                            <td>
                                                ${env.JOB_NAME}
                                            </td>
                                        </tr>

                                        <tr>
                                            <td>
                                                <b>Build Number</b>
                                            </td>

                                            <td>
                                                ${env.BUILD_NUMBER}
                                            </td>
                                        </tr>

                                        <tr>
                                            <td>
                                                <b>Stage</b>
                                            </td>

                                            <td>
                                                NPM Audit (Security Scan)
                                            </td>
                                        </tr>

                                        <tr>
                                            <td>
                                                <b>Status</b>
                                            </td>

                                            <td>
                                                ${env.SECURITY_STAGE_STATUS}
                                            </td>
                                        </tr>

                                        <tr>
                                            <td>
                                                <b>Build URL</b>
                                            </td>

                                            <td>
                                                <a href="${env.BUILD_URL}">
                                                    ${env.BUILD_URL}
                                                </a>
                                            </td>
                                        </tr>

                                    </table>

                                    <p>
                                        The npm audit security scan has
                                        completed.
                                    </p>

                                    <p>
                                        The stage-specific npm audit log and
                                        Jenkins console log are attached for
                                        review.
                                    </p>

                                    <p>
                                        <b>Interpretation:</b>
                                        A FAILURE status may indicate that
                                        npm audit successfully detected known
                                        vulnerabilities in the intentionally
                                        vulnerable nodejs-goof application.
                                    </p>

                                </body>
                                </html>
                            """,


                            /*
                             * Attach Jenkins Console Output.
                             */
                            attachLog: true,


                            /*
                             * Compress the Jenkins console log.
                             */
                            compressLog: true,


                            /*
                             * Attach the dedicated npm audit report.
                             */
                            attachmentsPattern:
                                'logs/npm-audit.log'
                        )

                        echo 'Security Scan notification email processed.'
                    }
                }
            }
        }
    }


    // =========================================================================
    // PIPELINE-LEVEL POST PROCESSING
    //
    // This block is separate from the two stage-level email post blocks.
    //
    // Its purpose is to archive the generated log files after the entire
    // pipeline has completed.
    // =========================================================================
    post {

        always {

            echo '=========================================================='
            echo 'PIPELINE POST-PROCESSING'
            echo '=========================================================='

            /*
             * Preserve all generated logs as Jenkins build artefacts.
             */
            archiveArtifacts(
                artifacts: 'logs/*.log',
                allowEmptyArchive: true
            )

            echo 'Generated logs archived as Jenkins build artefacts.'

            echo "Run Tests status: ${env.TEST_STAGE_STATUS}"

            echo """
                Coverage status:
                ${env.COVERAGE_STAGE_STATUS}
            """

            echo """
                Security Scan status:
                ${env.SECURITY_STAGE_STATUS}
            """

            echo 'Task 7.1C DevSecOps Pipeline execution completed.'
        }
    }
}
