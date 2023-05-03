#!/usr/bin/env groovy

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
                bat 'sfdx force:org:create -f config/project-scratch-def.json -a YourScratchOrgAlias -s'
            }
        }

        stage('Push Source') {
            steps {
                bat 'sfdx force:source:push -u YourScratchOrgAlias'
            }
        }

        stage('Run Tests') {
            steps {
                bat 'sfdx force:apex:test:run -u YourScratchOrgAlias -c -r human'
            }
        }

        stage('Delete Scratch Org') {
            steps {
                bat 'sfdx force:org:delete -u YourScratchOrgAlias -p'
            }
        }
    }
}
