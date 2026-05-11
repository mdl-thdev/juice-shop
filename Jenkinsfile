pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
    }
    // triggers {
    //     pollSCM('H/5 * * * *') // check every 5 minutes
    // }
    environment {
        // Fetches the installation path of the SonarQube Scanner configured in Jenkins and assigns it to this variable
        SONAR_SCANNER_HOME = tool 'SonarQube Scanner'
        // Pulls a secure token from Jenkins Credentials Manager (ID: snyk-token)
        SNYK_TOKEN         = credentials('snyk-token')
        // Sets a directory name where reports will be stored
        REPORT_DIR         = 'reports'
    }
    // Pipeline Options
    // Configures global pipeline behavior
    options {
        // Adds timestamps to console logs
        timestamps()
        // Stops the pipeline if it runs longer than 60 minutes
        timeout(time: 60, unit: 'MINUTES')
        // Keeps only the last 10 builds to save disk space
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    stages {
        stage('Checkout SCM') {
            steps {
                echo '>>> Checking out Juice Shop source...'
                git branch: 'mary',
                    url: 'https://github.com/mdl-thdev/juice-shop.git'
                sh "mkdir -p ${REPORT_DIR}" // Creates the reports directory (if it doesn't exist)
            }
        }

        stage('Verify Node'){
            steps {
                sh 'node --version' // Confirms NodeJS plugin is working correctly
                sh 'npm --version'
            }
        }

        stage('Install Dependencies') {
            steps {
                echo '>>> Installing dependencies (skipping postinstall to avoid Angular build)...'
                // Installs Node.js dependencies; --ignore-scripts avoids running post-install scripts (e.g., Angular build)
                sh 'npm install --ignore-scripts'
            }
        }

        // Runs Static Application Security Testing (SAST)
        stage('SAST - SonarQube Scan') {
            steps {
                echo '>>> Running SAST with SonarQube...'
                // Loads SonarQube server configuration from Jenkins
                withSonarQubeEnv('SonarQube') {
                    // Executes the SonarQube scanner
                    // Unique project identifier in SonarQube
                    // Display name in SonarQube UI
                    // Sets project version
                    // Tells SonarQube to scan the current directory
                    // Excludes node_modules, tests, compiled frontend assets, and Angular cache from scanning
                    // Points to TypeScript configuration file
                    // Provides test coverage report path
                    sh """
                        ${SONAR_SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=juice-shop \
                          -Dsonar.projectName='Juice Shop' \
                          -Dsonar.projectVersion=19.2.1 \
                          -Dsonar.sources=. \
                          -Dsonar.exclusions=**/node_modules/**,**/test/**,**/frontend/dist/**,**/frontend/src/assets/**,**/.angular/** \
                          -Dsonar.typescript.tsconfigPath=tsconfig.json \
                          -Dsonar.javascript.lcov.reportPaths=build/reports/coverage/server-tests/lcov.info \
                          -Dsonar.sourceEncoding=UTF-8
                    """
                } 
            }
        }

        // Checks SonarQube quality gate result
        stage('SAST - SonarQube Analysis') {
            steps {
                echo '>>> Checking SonarQube Quality Gate result...'
                // Waits max 5 minutes; Requires a SonarQube webhook configured in SonarQube pointing back to: http://<jenkins-url>/sonarqube-webhook/
                // Without this webhook, waitForQualityGate will hang until timeout
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false  // Waits for SonarQube result
                } // abortPipeline: false → pipeline continues even if it fails
            }
        }

        //Runs Software Composition Analysis (dependency vulnerability scan)
        stage('SCA - Snyk Scan') {
            steps {
                echo '>>> Running SCA with Snyk...'
                sh """
                    snyk auth \$SNYK_TOKEN

                    snyk test \
                        --all-projects \
                        --severity-threshold=low \
                        --json > ${REPORT_DIR}/snyk-report.json || true

                    snyk test \
                        --all-projects \
                        --severity-threshold=low || true

                    echo "Snyk scan complete."
                """
            } // Runs again for human-readable console output
        }   
    }
    post {
        // Runs regardless of success/failure
        always {
            echo '>>> Archiving scan reports...'
            // Saves report files as Jenkins build artifacts
            // snyk-report.json is already inside ${REPORT_DIR}/ so only one glob is needed
            archiveArtifacts artifacts: 'reports/**',
                             allowEmptyArchive: true

            echo 'Pipeline finished.'
        }
        success {
            echo 'All stages completed. Review findings in SonarQube and Snyk reports.'
        }
        failure {
            echo 'Pipeline failed. Check logs above for details.'
        }
    }
}
