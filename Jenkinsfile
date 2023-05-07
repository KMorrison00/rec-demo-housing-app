#!/usr/bin/env groovy
// helper function to be OS agnostic

String command(String script) {
    if (isUnix()) {
        return sh(returnSatust: true, script: script)
    }
    return bat(returnStatus: true, script: script)
}
String command_stdout(String script) {
    if (isUnix()) {
        return sh(returnStdout: true, script: script)
    }
    return bat(returnStdout: true, script: script).trim().readLines().drop(1).join(' ')
}

// set credentials for all the CI steps, env variables are set in jenkins ui
// private key is stored in credentials because its a file
pipeline {
    agent any

    environment {
        SF_CONSUMER_KEY = "${env.SF_CONSUMER_KEY}"
        SF_USERNAME = "${env.SF_USERNAME}"
        TEST_LEVEL = 'RunLocalTests'
        PACKAGE_NAME = 'test_package_1'
        SF_INSTANCE_URL = "${env.SF_INSTANCE_URL}"
        ALIAS = 'ciorg'
        MIN_REQUIRED_COVERAGE = 65.0
    }

    stages {
        // Check out code from source control based on pushes to pr branches.

        stage('Checkout Source') {
            steps {
                checkout scm
                git branch: env.BRANCH_NAME, credentialsId: 'Jenkins',
                 url: 'https://github.com/KMorrison00/rec-demo-housing-app'
            }
        }
        stage('Run Static Code Analysis') {
            steps {
                script {
                    // Install PMD and run static code analysis, saving the results as an XML file
                    command('pmd -d . -R rulesets/java/basic.xml -f xml > pmd-report.xml')
                }
            }
        }

        stage('Publish Static Code Analysis Results') {
            steps {
                script {
                    // Publish the PMD static code analysis results in Jenkins
                    recordIssues tool: pmdParser(pattern: 'pmd-report.xml')
                }
            }
        }

        // Authorize the Dev Hub org with JWT key and give it an alias.
        stage('Authorize DevHub And Create Scratch Org') {
            steps {
                script {
                    withCredentials([file(credentialsId: env.SERVER_KEY_CREDENTALS_ID, variable: 'server_key_file')]) {
                        command("sfdx force:auth:jwt:grant --instance-url ${SF_INSTANCE_URL} --client-id" +
                            " ${SF_CONSUMER_KEY} --username ${SF_USERNAME} --jwt-key-file $server_key_file" +
                            ' --set-default-dev-hub --alias HubOrg')
                        command("sfdx force:org:create --target-dev-hub HubOrg  "+
                                "--definitionfile config/project-scratch-def.json "+
                                "--setalias ${ALIAS} --wait 10 --durationdays 1")
                    }
                }
            }
        }

        // Display test scratch org info.
        stage('Display Scratch Org') {
            steps {
                script {
                    command("sfdx force:org:display --targetusername ${ALIAS}")
                }
            }
        }

        // Push source to test scratch org.
        stage('Push To Scratch Org') {
            steps {
                script {
                    command("sfdx force:source:push --targetusername ${ALIAS}")
                }
            }
        }

        // Run unit tests in test scratch org.
        stage('Run Tests In Scratch Org') {
            steps {
                script {
                    def apexTestFile = 'test_results/apex_results.txt'
                    def filePipe = '^>'
                    if (isUnix()) {
                        filePipe = '>'
                    }
                    command('if not exist test_results mkdir test_results')

                    command_stdout("sfdx force:apex:test:run --target-org ${ALIAS} " +
                        "--code-coverage --result-format human --test-level ${TEST_LEVEL} " +
                        "--wait 10 ${filePipe} ${apexTestFile}")

                    archiveArtifacts artifacts: apexTestFile
                    // check coverage results
                    def fileContent = readFile apexTestFile
                    def lines = fileContent.readLines()

                    lines.each { line ->
                        if (line.contains('Org Wide Coverage')) {
                            def coverageStr = line.split()[3]
                            def coverage = Double.valueOf(coverageStr.trim().replace('%', ''))
                            try {
                                // Your pipeline code, including the coverage check
                                if (coverage >= Double.valueOf(MIN_REQUIRED_COVERAGE)) {
                                    echo "Coverage is ${coverage}%"
                                } else {
                                    error "Coverage is below minimum threshold of ${MIN_REQUIRED_COVERAGE}"
                                }
                            } catch (Exception e) {
                                // Handle the exception and display the error message
                                currentBuild.result = 'FAILURE'
                                currentBuild.description = "Error: ${e.message}"
                            }
                        }
                    }
                }
            }
        }

        // Create package version.
        stage('Create Package Version') {
            steps {
                script {
                    output = command("sfdx force:package:version:create --package ${PACKAGE_NAME}"+
                                    ' --installationkeybypass --wait 10 --json --targetdevhubusername HubOrg')

                    // Wait 5 minutes for package replication.
                    sleep 300

                    def jsonSlurper = new groovy.json.JsonSlurper()
                    def response = jsonSlurper.parseText(output)

                    PACKAGE_VERSION = response.result.SubscriberPackageVersionId

                    response = null

                    echo "${PACKAGE_VERSION}"
                }
            }
        }
    }
    // post {
    //     always {
    //         script {
    //             // cleanup
    //             command("sfdx force:org:delete --targetusername ${ALIAS} --noprompt")
    //         }
    //     }
    // }
}

