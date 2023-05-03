#!/usr/bin/env groovy

def genericSh(cmd) {
    if (isUnix()) {
        sh cmd
    } else {
        bat cmd
    }
}

pipeline { 
    agent any 

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                git branch: env.BRANCH_NAME, credentialsId: 'Jenkins', url: 'https://github.com/KMorrison00/rec-demo-housing-app'
            }
        }

        stage('Create Scratch Org') {
            steps {
                genericSh('sfdx force:org:create -f config/project-scratch-def.json -a YourScratchOrgAlias -s')
            }
        }

        stage('Push Source') {
            steps {
                genericSh('sfdx force:source:push -u YourScratchOrgAlias')
            }
        }

        stage('Run Tests') {
            steps {
                genericSh('sfdx force:apex:test:run -u YourScratchOrgAlias -c -r human')
            }
        }

        stage('Delete Scratch Org') {
            steps {
                genericSh('sfdx force:org:delete -u YourScratchOrgAlias -p')
            }
        }
    }
}
