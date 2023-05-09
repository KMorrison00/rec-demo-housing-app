#!/usr/bin/env groovy

// helper functions to be OS agnostic
String command(String script) {
    if (isUnix()) {
        return sh(returnStatus: true, script: script)
    }
    return bat(returnStatus: true, script: script)
}
String commandStdout(String script) {
    if (isUnix()) {
        return sh(returnStdout: true, script: script)
    }
    return bat(returnStdout: true, script: script).trim().readLines().drop(1).join(' ')
}

String getFilePipe() {
    if (isUnix()) {
        return '>'
    }
    return '^>'
}

// used for passing package data between stages
String packageId  = ''
String packageName = ''

pipeline {
    agent any

    // set credentials for all the CI steps, env variables are set in jenkins ui
    // private key is stored in credentials because its a file
    environment {
        SF_CONSUMER_KEY = "${env.SF_CONSUMER_KEY2}"
        SF_USERNAME = "${env.SF_USERNAME2}"
        TEST_LEVEL = 'RunAllTestsInOrg'
        SF_INSTANCE_URL = "${env.SF_INSTANCE_URL}"
        SCRATCH_ORG_ALIAS = 'scratch_org'
        HUB_ORG = 'ciorg'
        MIN_REQUIRED_COVERAGE = 65.0
        GITHUB_TOKEN = credentials('kmorrison00-github-creds')
    }

    stages {
        // Check out code from source control based on pushes to pr branches.

        stage('Checkout Source') {
            steps {
                checkout scm
                git branch: env.BRANCH_NAME, credentialsId: 'kmorrison00-github-creds',
                 url: 'https://github.com/KMorrison00/rec-demo-housing-app'
            }
        }
        stage('Run Static Code Analysis') {
            steps {
                script {
                    // Install PMD and run static code analysis, saving the results as an XML file
                    command('pmd check -d src/main/default -R pmd-ruleset.xml' +
                        ' --no-fail-on-violation --format=xml -r pmd-report.xml')
                }
            }
        }

        // Publish the PMD static code analysis results in Jenkins viewable via warnings extension
        stage('Publish Static Code Analysis Results') {
            steps {
                script {
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
                            " ${SF_CONSUMER_KEY} --username ${SF_USERNAME} --jwt-key-file ${server_key_file}" +
                            " --set-default-dev-hub --alias ${HUB_ORG}")
                        // // delete old scratch org 
                        // command("sfdx force:org:delete --target-dev-hub ${SCRATCH_ORG_ALIAS} --noprompt")
                        // // create new one
                        // command("sfdx force:org:create --target-dev-hub ${HUB_ORG} "+
                        //         '--definitionfile config/project-scratch-def.json '+
                        //         "--setalias ${SCRATCH_ORG_ALIAS} --wait 10 --durationdays 1")
                    }
                }
            }
        }
        // Display test scratch org info.
        stage('Display Scratch Org') {
            steps {
                script {
                    def filePipe = getFilePipe()
                    command("sfdx force:org:display --target-org ${SCRATCH_ORG_ALIAS} ${filePipe} org_details.txt")
                    archiveArtifacts "org_details.txt"
                }
            }
        }

        // // Push source to test scratch org.
        // stage('Push To Scratch Org') {
        //     steps {
        //         script {
        //             command("sfdx force:source:push --target-org ${SCRATCH_ORG_ALIAS}")
        //         }
        //     }
        // }

        // Run unit tests in test scratch org.
        stage('Run Tests In Scratch Org') {
            steps {
                script {
                    def apexTestFile = 'test_results/apex_results.txt'
                    command('if not exist test_results mkdir test_results')
                    def filePipe = getFilePipe()
                    commandStdout("sfdx apex:run:test --target-org ${SCRATCH_ORG_ALIAS} " +
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
        stage('Packaging') {
            stages {
                // check for a package that exists so we can create or update it
                stage('Check Package') {
                    steps {
                        script {
                            def output = commandStdout("sfdx force:package:list --target-dev-hub ${HUB_ORG} --json")
                            def response = readJSON text: output
                            echo response.toString()
                            def packageExists = false
                            def sfdxProject = readJSON file: 'sfdx-project.json'
                            packageName = sfdxProject.packageDirectories[0].package
                            try {
                                echo "checking if package exists"
                                for (def pkg in response.result) {
                                    if (pkg.Name == packageName) {
                                        packageExists = true
                                        echo "Package: ${packageName} Found"
                                    }
                                
                                }
                            } catch (Exception e) {
                                echo "Package Name not found"
                            }
                            echo "package exists: ${packageExists}"
                            if (packageExists == true) {
                                echo "updating sdfx-project.json with alias ${packageName}"
                                // update sdfx-project.json file for later steps
                                packageId = response.result[0].Id
                                sfdxProject.packageAliases[packageName] = packageId
                                writeJSON file: 'sfdx-project.json', json: sfdxProject
                            } 
                        }
                    }
                }

                // if theres no package yet, create one
                stage('Create New Package') {
                    when {
                        expression { 
                            packageId == ''
                        }
                    }
                    steps {
                        script {
                            output = commandStdout("sfdx package:create --name ${packageName}" +
                                " --package-type Unlocked --target-dev-hub ${HUB_ORG} --path src --json")
                            def response = readJSON text: output
                            echo "Created new package with ID: ${response.result.Id}"
                        }
                    }
                }
                // Create package version.
                stage('Create Package version') {
                    when {
                        expression { 
                            packageId.contains('0Ho')
                        }
                    }
                    steps {
                        script {
                            output = commandStdout("sfdx force:package:version:create --package ${packageId}" +
                                    " --installation-key-bypass --wait 10 --json --target-dev-hub ${HUB_ORG}" + 
                                    " --definitionfile config/project-scratch-def.json")
                            def response = readJSON text: output
                            echo response.toString()
                            // echo "Updated package with ID: ${response.result.Id}"
                        }
                    }
                }
            }
        }
    }
}
