#!/usr/bin/env groovy

import java.util.regex.Pattern
import java.util.regex.Matcher

// helper function to be OS agnostic
String command(String script) {
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
        TEST_LEVEL = 'RunAllTestsInOrg'
        PACKAGE_NAME = 'test_package_1'
        SF_INSTANCE_URL = "${env.SF_INSTANCE_URL}"
        ALIAS = 'ciorg'
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

        // Authorize the Dev Hub org with JWT key and give it an alias.
        stage('Authorize DevHub And Create Scratch Org') {
            steps {
                script {
                    withCredentials([file(credentialsId: env.SERVER_KEY_CREDENTALS_ID, variable: 'server_key_file')]) {
                        command("sfdx force:auth:jwt:grant --instance-url ${SF_INSTANCE_URL} --client-id" +
                            " ${SF_CONSUMER_KEY} --username ${SF_USERNAME} --jwt-key-file $server_key_file" +
                            ' --set-default-dev-hub --alias HubOrg')
                    // command("sfdx force:org:create --target-dev-hub HubOrg  "+
                    //          "--definitionfile config/project-scratch-def.json "+
                    //          "--setalias ${ALIAS} --wait 10 --durationdays 1")
                    }
                }
            }
        }

        // // Display test scratch org info.
        // stage('Display Scratch Org') {
        //     steps {
        //         script {
        //             command("sfdx force:org:display --targetusername ${ALIAS}")
        //         }
        //     }
        // }

        // // Push source to test scratch org.
        // stage('Push To Scratch Org') {
        //     steps {
        //         script {
        //             command("sfdx force:source:push --targetusername ${ALIAS}")
        //         }
        //     }
        // }

        // Run unit tests in test scratch org.
        stage('Run Tests In Scratch Org') {
            steps {
                script {
                    String rtnMsg = command("sfdx force:apex:test:run --targetusername ${ALIAS}" +
                        " --resultformat json --codecoverage --testlevel ${TEST_LEVEL}")
                    println(rtnMsg)
                    // looks for the -i char in rtnMsg indicating the testrunid and then grabs the next arg
                    // which is the testrunid
                    Pattern pattern = /-i\s+(\S+)/
                    Matcher matcher = (rtnMsg =~ pattern)
                    String match = ''
                    if (matcher.find()) {
                        match = matcher.group(0)[1] // Store the matched string in a serializable object
                    }
                    String testRunId = match
                    println("Test Run ID: ${testRunId}")
                    command("sfdx force:apex:test:report --targetusername ${ALIAS} --resultformat junit " +
                        "--codecoverage --testrunid ${testRunId} --outputdir test_results")
                    archiveArtifacts artifacts: 'test_results/*'
                }
            }
        }
    // Delete test scratch org.
    // stage('Delete Scratch Org') {
    //     steps {
    //         script {
    //             command("sfdx force:org:delete --targetusername ${ALIAS} --noprompt")
    //         }
    //     }
    // }
    }
// post {
//     failure {
//         script {
//             command("sfdx force:org:delete --targetusername ${ALIAS} --noprompt")
//         }
//     }
// }
}

        // Create package version.
        // stage('Create Package Version') {
        //     steps {
        //         script {
        //             output = command("sfdx force:package:version:create --package ${PACKAGE_NAME}"+
        //                              " --installationkeybypass --wait 10 --json --targetdevhubusername HubOrg")

        //             // Wait 5 minutes for package replication.
        //             sleep 300

        //             def jsonSlurper = new groovy.json.JsonSlurper
        //             def response = jsonSlurper.parseText(output)

        //             PACKAGE_VERSION = response.result.SubscriberPackageVersionId

        //             response = null

        //             echo ${PACKAGE_VERSION}
        //         }
        //     }
        // }

        // // Create new scratch org to install package to.
        // stage('Create Package Install Scratch Org') {
        //     steps {
        //         script {
        //             command("sfdx force:org:create --targetdevhubusername HubOrg --setdefaultusername"+
        //                  " --definitionfile config/project-scratch-def.json "+
        //                  "--setalias installorg --wait 10 --durationdays 1")
        //         }
        //     }
        // }

        // // Display install scratch org info.
        // stage('Display Install Scratch Org') {
        //     steps {
        //         script {
        //             command("sfdx force:org:display --targetusername installorg")
        //         }
        //     }
        // }

        // // Install package in scratch org.
        // stage('Install Package In Scratch Org') {
        //     steps {
        //         script {
        //             command("sfdx force:package:install --package ${PACKAGE_VERSION}"+
        //                      ' --targetusername installorg --wait 10')
        //         }
        //     }
        // }

        // // Run unit tests in package install scratch org.
        // stage('Run Tests In Package Install Scratch Org') {
        //     steps {
        //         script {
        //             command("sfdx force:apex:test:run --targetusername installorg"+
        //              " --resultformat tap --codecoverage --testlevel ${TEST_LEVEL} --wait 10")
        //         }
        //     }
        // }

        // // Delete package install scratch org.
        // stage('Delete Package Install Scratch Org') {
        //     steps {
        //         script {
        //             command("sfdx force:org:delete --targetusername installorg --noprompt")
        //         }
        //     }
        // }
    // }
// }

