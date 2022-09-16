// Import Shared Library
@Library('my-shared-library') _ // Imports entire library, alt specify tag or branch @Library('my-shared-library@1.0') _

def sc = new org.slib.sonarqube()

def globalVar = '' // Enables var to be used throughout pipeline


pipeline {
    agent { label 'linux' } // or agent any
    parameters {
        string(name: 'slackChannel', defaultValue: 'devops', description: 'Slack channel to report Jenkins notifications')
        string(name: 'jiraProject', defaultValue: 'SAN', description: 'Key for the Jira Project to update')
        string(name: 'giturl', defaultValue: 'https://github.com/chrisj-au/Jenkinsfile-demo.git', description: 'Source code location')
        string(name: 'scProjectName', defaultValue: "${env.JOB_BASE_NAME}", description: 'SonarQube Project Name')
        string(name: 'scProjectKey', defaultValue: "${env.JOB_BASE_NAME.replaceAll('\\s','')}", description: 'SonarQube Project Key')
        string(name: 'scFolderExclusions', defaultValue: '', description: 'SonarQube Folder Exclusions (comma seperated)')
    }
    environment {
        JIRA_SITE="${jiraName}" // This var is automatically used by the jira-steps plugin
    }
    options {
        timestamps() // plugin: https://plugins.jenkins.io/timestamper/
    }
    stages {
        stage('Initial Tasks') {
            steps {
                 script {
                    checkoutSCM // Uses repo from job configuration
                    startedBy = whoStartedJob() // Record what/who triggered pipeline (by user, git trigger, schedule etc)
                    echo "Pipeline triggered by: ${startedBy}"
                }
            }
        }
        stage('Blocking Step') {
            when {
                expression {
                    input message: 'Tag on Docker Hub?'
                    // if input is Aborted, the whole build will fail, otherwise
                    // we must return true to continue
                    return true
                }
                beforeAgent true
            }
            steps {
                
            }
        }
        stage('Set Env Vars') {
            steps {
                script {
                    env.IS_NEW_VERSION = "YES"
                }
            }
        }
        stage('More Conditionals') {
            when {
                branch "master"
                environment name: "IS_NEW_VERSION", value: "YES"
            }
            
        }
        stage("A stage to be skipped") {
            when {
                expression { false }  //skip this stage
            }
            steps {
                echo 'This step will never be run'
            }
        }
    }
    post {
        always {
            cleanWs()   // Clean up Jenkins workspace - this is a plugin
        }
        fixed {
            echo "Pipeline is now successful after previous failure"
        }
        success {
            echo "Build Successful"
        }
        failure {
            script { 
                echo "Build failed"
                // Create Jira?
                // Send Slack?
            }
        }
        aborted {
            // This stage is triggered due to a non-error termination
            // e.g. timeout being exceeded, notifications to DevOps team go here
            echo 'aborted'
        }
    }
}
