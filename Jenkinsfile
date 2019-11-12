def sc = new org.slib.sonarqube()
def git = new org.slib.git()
def slack = new org.slib.slack()
def jira = new org.slib.jiraUpdate()

def gitCredsId = '' // GitHub Credentials Id
def sonarName = 'SonarQube Beta Server' // Server registered in Jenkins
def jiraName = 'DevOps JIRA'    // Server registered in Jenkins

pipeline {
    agent any
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
    stages {
        stage('GIT: Checkout Source') {
            steps {
                 script {
                    failedStage=env.STAGE_NAME
                    gitRepo = git.checkOut(params.giturl, gitCredsId, 'master')
                    gitTag = git.getTags()  // Retrieve commit tag
                    gitAuthorEmail = git.getDetails() // Retrieve author details from commit
                    echo "Branch: ${env.BRANCH_NAME}"
                    startedBy = whoStartedJob() // Record what/who triggered pipeline (by user, git trigger, schedule etc)
                    echo "Pipeline triggered by: ${startedBy}"
                }
            }
        }
        stage('SonarQube: Trigger Scan') {
            steps {
                withSonarQubeEnv(sonarName) { // await sonarqube response (requires SC to be configured with Jenkins generic hook)
                    script {
                        failedStage=env.STAGE_NAME
                        sc.RunSonarQube(
                            params.scProjectKey,
                            params.scProjectName,
                            params.scFolderExclusions,
                            env.WORKSPACE,
                            env.BUILD_NUMBER,
                            "staging")
                    }
                }
            }
        }
        stage('SonarQube: Evaluate Quality Gate') {
            agent none
            options { 
                timestamps()
                timeout(time: 1, unit: 'HOURS') }
            steps {
                script {
                    failedStage=env.STAGE_NAME
                    waitForQualityGate abortPipeline: true  // Wait for scan result and abort pipeline on failure
                }
            }
        }
    }
    post {
        always {
            cleanWs()   // Clean up Jenkins workspace
        }
        fixed {
            echo "Pipeline is now successful after previous failure"
            script {
                jira.NewComment(currentBuild.previousBuild.buildVariables.jiraIssue, "Build Successfully completeted, see [${BUILD_ID}|${env.BUILD_URL}]")
                slackSend color: "#00ff00", channel: slackChannel, message: "Closing Issue: ${currentBuild.previousBuild.buildVariables.jiraIssue}"
                // ToDo: Close jira issue
            }
        }
        success {
            script {
                script { slack.sendBuildResult(slackChannel, startedBy, gitAuthorEmail, gitTag) }
            }
        }
        failure {
            script { 
                def jiraInfo = jiraGetServerInfo() // Retrieve Jira server info to get baseurl
                jira.LogResult(jiraProject, gitAuthorEmail, startedBy, failedStage)
                slack.sendBuildResult(slackChannel, startedBy, gitAuthorEmail, gitTag, failedStage, jiraInfo.data.baseUrl)
            }
        }
        aborted {
            // This stage is triggered due to a non-error termination
            // e.g. timeout being exceeded, notifications to DevOps team go here
            echo 'aborted'
            script { slack.sendBuildResult(slackChannel, startedBy, gitAuthorEmail, gitTag, failedStage, jiraInfo.data.baseUrl) }
        }
    }
}