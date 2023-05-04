#!/usr/bin/env groovy
import groovy.json.JsonSlurperClassic

// helper function to be OS agnostic
def command(script) {
    if (isUnix()) {
        return sh(returnStatus: true, script: script);
    } else {
        return bat(returnStatus: true, script: script);
    }
}

// set credentials for all the CI steps, env variables are set in jenkins ui
// private key is stored in credentials because its a file
pipeline {
    agent any

    environment {
        SF_CONSUMER_KEY="${env.SF_CONSUMER_KEY}"
        SF_USERNAME="${env.SF_USERNAME}"
        TEST_LEVEL='RunLocalTests'
        PACKAGE_NAME='test_package_1'
        SF_INSTANCE_URL = "${env.SF_INSTANCE_URL}"
        toolbelt = tool 'toolbelt'
    }

    stages {
        // Check out code from source control based on pushes to pr branches.

        stage('checkout source') {
            steps {
                checkout scm
                git branch: env.BRANCH_NAME, credentialsId: 'Jenkins', url: 'https://github.com/KMorrison00/rec-demo-housing-app'
            }
        }

        // Authorize the Dev Hub org with JWT key and give it an alias.
        stage('Authorize DevHub') {
            steps {
                script {
                    // command("rm -rf $HOME/.sfdx")
                    withCredentials([file(credentialsId: env.SERVER_KEY_CREDENTALS_ID, variable: 'server_key_file')]) {
                        // try logging out prio to reset credentials?
                        command("${toolbelt}/sfdx force:auth:logout --noprompt --username ${SF_USERNAME}")
                        rc = command("${toolbelt}/sfdx force:auth:jwt:grant --instance-url ${SF_INSTANCE_URL} --client-id ${SF_CONSUMER_KEY} --username ${SF_USERNAME} --jwt-key-file $server_key_file --set-default-dev-hub --alias HubOrg")
                        if (rc != 0) {
                            error 'Salesforce dev hub org authorization failed.'
                        }
                    }
                }
            }
        }

        // Create new scratch org to test your code.
        stage('Create Test Scratch Org') {
            steps {
                script {
                    rc = command("${toolbelt}/sfdx force:org:create --target-dev-hub HubOrg  --definitionfile config/project-scratch-def.json --setalias ciorg --wait 10 --durationdays 1")
                    if (rc != 0) {
                        error 'Salesforce test scratch org creation failed.'
                    }
                }
            }
        }

        // Display test scratch org info.
        stage('Display Test Scratch Org') {
            steps {
                script {
                    rc = command("${toolbelt}/sfdx force:org:display --targetusername ciorg")
                    if (rc != 0) {
                        error 'Salesforce test scratch org display failed.'
                    }
                }
            }
        }

        // Push source to test scratch org.
        stage('Push To Test Scratch Org') {
            steps {
                script {
                    rc = command("${toolbelt}/sfdx force:source:push --targetusername ciorg")
                    if (rc != 0) {
                        error 'Salesforce push to test scratch org failed.'
                    }
                }
            }
        }

        // Run unit tests in test scratch org.
        stage('Run Tests In Test Scratch Org') {
            steps {
                script {
                    rc = command("${toolbelt}/sfdx force:apex:test:run --targetusername ciorg --wait 10 --resultformat tap --codecoverage --testlevel ${TEST_LEVEL}")
                    if (rc != 0) {
                        error 'Salesforce unit test run in test scratch org failed.'
                    }
                }
            }
            
        }

        // Delete test scratch org.
        stage('Delete Test Scratch Org') {
            steps {
                script {
                    rc = command("${toolbelt}/sfdx force:org:delete --targetusername ciorg --noprompt")
                    if (rc != 0) {
                        error 'Salesforce test scratch org deletion failed.'
                    }
                }
            }
        }

        // Create package version.
        stage('Create Package Version') {
            steps {
                script {
                    if (isUnix()) {
                        output = sh returnStdout: true, script: "${toolbelt}/sfdx force:package:version:create --package ${PACKAGE_NAME} --installationkeybypass --wait 10 --json --targetdevhubusername HubOrg"
                    } else {
                        output = bat(returnStdout: true, script: "${toolbelt}/sfdx force:package:version:create --package ${PACKAGE_NAME} --installationkeybypass --wait 10 --json --targetdevhubusername HubOrg").trim()
                        output = output.readLines().drop(1).join(" ")
                    }

                    // Wait 5 minutes for package replication.
                    sleep 300

                    def jsonSlurper = new JsonSlurperClassic()
                    def response = jsonSlurper.parseText(output)

                    PACKAGE_VERSION = response.result.SubscriberPackageVersionId

                    response = null

                    echo ${PACKAGE_VERSION}
                }
            }
            
        }

        // Create new scratch org to install package to.
        stage('Create Package Install Scratch Org') {
            steps {
                script {
                    rc = command("${toolbelt}/sfdx force:org:create --targetdevhubusername HubOrg --setdefaultusername --definitionfile config/project-scratch-def.json --setalias installorg --wait 10 --durationdays 1")
                    if (rc != 0) {
                        error 'Salesforce package install scratch org creation failed.'
                    }
                }
            }
        }

        // Display install scratch org info.
        stage('Display Install Scratch Org') {
            steps {
                script {
                    rc = command("${toolbelt}/sfdx force:org:display --targetusername installorg")
                    if (rc != 0) {
                        error 'Salesforce install scratch org display failed.'
                    }
                }
            }
        }

        // Install package in scratch org.
        stage('Install Package In Scratch Org') {
            steps {
                script {
                    rc = command("${toolbelt}/sfdx force:package:install --package ${PACKAGE_VERSION} --targetusername installorg --wait 10")
                    if (rc != 0) {
                        error 'Salesforce package install failed.'
                    }
                }
            }
        }

        // Run unit tests in package install scratch org.
        stage('Run Tests In Package Install Scratch Org') {
            steps {
                script {
                    rc = command("${toolbelt}/sfdx force:apex:test:run --targetusername installorg --resultformat tap --codecoverage --testlevel ${TEST_LEVEL} --wait 10")
                    if (rc != 0) {
                        error 'Salesforce unit test run in pacakge install scratch org failed.'
                    }
                }
            }
        }

        // Delete package install scratch org.
        stage('Delete Package Install Scratch Org') {
            steps {
                script {
                    rc = command("${toolbelt}/sfdx force:org:delete --targetusername installorg --noprompt")
                    if (rc != 0) {
                        error 'Salesforce package install scratch org deletion failed.'
                    }
                }
            }
        }
    }
}

