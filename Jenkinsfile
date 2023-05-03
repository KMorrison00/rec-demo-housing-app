#!/usr/bin/env groovy

def run_cmd(cmd) {
    if (isUnix()) {
        sh cmd
    } else {
        bat cmd
    }
}

// set credentials for all the CI steps, env variables are set in jenkins ui
// private key is stored in credentials because its a file
pipeline { 
    agent any 

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                git branch: env.BRANCH_NAME, credentialsId: 'Jenkins', url: 'https://github.com/KMorrison00/rec-demo-housing-app'
            }
        }
        
        stage('Authorize Dev Hub') {
            steps {
                withCredentials([file(credentialsId: 'salesforce_private_key', variable: 'DEVHUB_PRIVATE_KEY_FILE')]) {
                    run_cmd("sfdx force:auth:jwt:grant --clientid \\${env.CONSUMER_KEY} --jwtkeyfile \\${DEVHUB_PRIVATE_KEY_FILE} --username \\${env.SALESFORCE_USERNAME} --setdefaultdevhubusername --setalias HubOrg")
                }
            }
        }

        stage('Create Scratch Org') {
            steps {
                run_cmd('sfdx force:org:create -f config/project-scratch-def.json -a YourScratchOrgAlias -s')
            }
        }

        stage('Push Source') {
            steps {
                run_cmd('sfdx force:source:push -u YourScratchOrgAlias')
            }
        }

        stage('Run Tests') {
            steps {
                run_cmd('sfdx force:apex:test:run -u YourScratchOrgAlias -c -r human')
            }
        }

        stage('Delete Scratch Org') {
            steps {
                run_cmd('sfdx force:org:delete -u YourScratchOrgAlias -p')
            }
        }
    }
}