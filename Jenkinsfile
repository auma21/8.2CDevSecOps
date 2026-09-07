pipeline {
    agent any

    // ========================================================================
    // PIPELINE OPTIONS
    // ========================================================================
    options {
        /*
         * The source repository is explicitly checked out in the Checkout
         * stage, so Jenkins' normal workspace checkout is disabled.
         */
        skipDefaultCheckout(true)

        /*
         * Adds timestamps to console messages, improving the audit trail.
         */
        timestamps()
    }

    // ========================================================================
    // AUTOMATIC GITHUB POLLING
    // ========================================================================
    triggers {
        /*
         * Jenkins periodically polls the GitHub repository for new commits.
         * The task permits scheduled polling instead of a GitHub webhook.
         */
        pollSCM('H/2 * * * *')
    }

    // ========================================================================
    // PIPELINE VARIABLES
    // ========================================================================
    environment {
        REPO_URL = 'https://github.com/auma21/8.2CDevSecOps.git'

        EMAIL_TO = 'aumarbles@gmail.com'

        TEST_STAGE_STATUS = 'NOT_RUN'

        SECURITY_STAGE_STATUS = 'NOT_RUN'
    }

    stages {

        // ====================================================================
        // STAGE 1: CHECKOUT
        // Task: Retrieve the application source code.
        // Tool: Git / GitHub
        // ====================================================================
        stage('Checkout') {
            steps {
                echo 'Checking out nodejs-goof from GitHub...'

                git branch: 'main',
                    url: "${env.REPO_URL}"
            }
        }

        // ====================================================================
        // STAGE 2: INSTALL DEPENDENCIES
        // Task: Install dependencies declared by the Node.js application.
        // Tool: npm
        // ====================================================================
        stage('Install Dependencies') {
            steps {
                echo 'Verifying Node.js and npm...'

                bat 'node --version'
                bat 'npm --version'

                echo 'Installing project dependencies...'

                bat 'npm install'
            }
        }

        // ====================================================================
        // STAGE 3: RUN TESTS
        // Task: Execute the test command configured in package.json.
        // Tool: npm / Snyk CLI as currently invoked by nodejs-goof
        // ====================================================================
        stage('Run Tests') {
            steps {
                script {

                    echo 'Running npm test...'

                    // Create the log directory when required.
                    bat 'if not exist logs mkdir logs'

                    /*
                     * returnStatus prevents a non-zero npm exit code from
                     * terminating the complete Jenkins Pipeline.
                     *
                     * The command output is written to npm-test.log.
                     */
                    int testExitCode = bat(
                        returnStatus: true,
                        script: '''
                            @echo off
                            npm test > logs\\npm-test.log 2>&1
                        '''
                    )

                    /*
                     * Determine the real result of the test operation.
                     */
                    if (testExitCode == 0) {
                        env.TEST_STAGE_STATUS = 'SUCCESS'

                        echo 'Run Tests completed successfully.'
                    } else {
                        env.TEST_STAGE_STATUS = 'FAILURE'

                        echo "Run Tests returned exit code ${testExitCode}."

                        /*
                         * Mark this particular stage as failed while allowing
                         * the overall Pipeline to continue.
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

                    // Display the saved test output in Jenkins Console Output.
                    bat 'type logs\\npm-test.log'
                }
            }

            // ================================================================
            // EMAIL 1: TEST-STAGE NOTIFICATION
            // ================================================================
            post {
                always {
                    emailext(
                        to: "${env.EMAIL_TO}",

                        subject:
                            "[Jenkins] Run Tests ${env.TEST_STAGE_STATUS} - " +
                            "${env.JOB_NAME} #${env.BUILD_NUMBER}",

                        mimeType: 'text/html',

                        body: """
                            <h2>Jenkins Test Stage Notification</h2>

                            <p>
                                <b>Job:</b>
                                ${env.JOB_NAME}
                            </p>

                            <p>
                                <b>Build Number:</b>
                                ${env.BUILD_NUMBER}
                            </p>

                            <p>
                                <b>Stage:</b>
                                Run Tests
                            </p>

                            <p>
                                <b>Status:</b>
                                ${env.TEST_STAGE_STATUS}
                            </p>

                            <p>
                                <b>Build URL:</b>
                                ${env.BUILD_URL}
                            </p>

                            <p>
                                The Run Tests stage has completed.
                                The stage-specific test log and Jenkins
                                console log are attached for review.
                            </p>
                        """,

                        /*
                         * Attach the complete Jenkins console log.
                         */
                        attachLog: true,

                        /*
                         * Compress the full Jenkins log to reduce attachment
                         * size.
                         */
                        compressLog: true,

                        /*
                         * Attach the log produced specifically by npm test.
                         */
                        attachmentsPattern: 'logs/npm-test.log'
                    )
                }
            }
        }

        // ====================================================================
        // STAGE 4: GENERATE COVERAGE REPORT
        // Task: Attempt to generate the requested coverage information.
        // Tool: npm
        // ====================================================================
        stage('Generate Coverage Report') {
            steps {
                script {

                    echo 'Attempting to generate coverage report...'

                    bat 'if not exist logs mkdir logs'

                    /*
                     * The assessment specifies that the Pipeline should
                     * continue even if this command does not succeed.
                     */
                    int coverageExitCode = bat(
                        returnStatus: true,
                        script: '''
                            @echo off
                            npm run coverage > logs\\coverage.log 2>&1
                        '''
                    )

                    if (coverageExitCode == 0) {
                        echo 'Coverage command completed successfully.'
                    } else {
                        echo """
                        Coverage command returned exit code
                        ${coverageExitCode}. Pipeline will continue as
                        required by the assessment.
                        """
                    }

                    bat 'type logs\\coverage.log'
                }
            }
        }

        // ====================================================================
        // STAGE 5: NPM AUDIT SECURITY SCAN
        // Task: Identify known dependency vulnerabilities.
        // Tool: npm audit
        // ====================================================================
        stage('NPM Audit (Security Scan)') {
            steps {
                script {

                    echo 'Running npm audit security scan...'

                    bat 'if not exist logs mkdir logs'

                    int auditExitCode = bat(
                        returnStatus: true,
                        script: '''
                            @echo off
                            npm audit > logs\\npm-audit.log 2>&1
                        '''
                    )

                    /*
                     * npm audit normally returns a non-zero exit code when
                     * vulnerabilities meeting its failure threshold exist.
                     */
                    if (auditExitCode == 0) {
                        env.SECURITY_STAGE_STATUS = 'SUCCESS'

                        echo 'NPM Audit completed without a failing result.'
                    } else {
                        env.SECURITY_STAGE_STATUS = 'FAILURE'

                        echo "NPM Audit returned exit code ${auditExitCode}."

                        /*
                         * Show the stage as failed while still completing
                         * remaining Pipeline post-processing.
                         */
                        catchError(
                            buildResult: 'SUCCESS',
                            stageResult: 'FAILURE'
                        ) {
                            error(
                                "Security scan identified issues. " +
                                "Exit code: ${auditExitCode}."
                            )
                        }
                    }

                    // Display the vulnerability results in Jenkins.
                    bat 'type logs\\npm-audit.log'
                }
            }

            // ================================================================
            // EMAIL 2: SECURITY-SCAN NOTIFICATION
            // ================================================================
            post {
                always {
                    emailext(
                        to: "${env.EMAIL_TO}",

                        subject:
                            "[Jenkins] Security Scan " +
                            "${env.SECURITY_STAGE_STATUS} - " +
                            "${env.JOB_NAME} #${env.BUILD_NUMBER}",

                        mimeType: 'text/html',

                        body: """
                            <h2>Jenkins Security Scan Notification</h2>

                            <p>
                                <b>Job:</b>
                                ${env.JOB_NAME}
                            </p>

                            <p>
                                <b>Build Number:</b>
                                ${env.BUILD_NUMBER}
                            </p>

                            <p>
                                <b>Stage:</b>
                                NPM Audit (Security Scan)
                            </p>

                            <p>
                                <b>Status:</b>
                                ${env.SECURITY_STAGE_STATUS}
                            </p>

                            <p>
                                <b>Build URL:</b>
                                ${env.BUILD_URL}
                            </p>

                            <p>
                                The npm audit security scan has completed.
                                The stage-specific audit log and Jenkins
                                console log are attached for review.
                            </p>
                        """,

                        attachLog: true,

                        compressLog: true,

                        attachmentsPattern: 'logs/npm-audit.log'
                    )
                }
            }
        }
    }

    // ========================================================================
    // PIPELINE POST-PROCESSING
    // ========================================================================
    post {
        always {

            /*
             * Retain the generated logs within the Jenkins build as
             * downloadable build artefacts.
             */
            archiveArtifacts(
                artifacts: 'logs/*.log',
                allowEmptyArchive: true
            )

            echo 'Task 7.1C DevSecOps Pipeline execution completed.'
        }
    }
}
