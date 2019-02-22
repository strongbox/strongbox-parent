@Library('jenkins-shared-libraries') _

def SERVER_ID  = 'carlspring-oss-snapshots'
def DEPLOY_SERVER_URL = 'https://repo.carlspring.org/content/repositories/carlspring-oss-snapshots/'
def PR_SERVER_URL = 'https://repo.carlspring.org/content/repositories/carlspring-oss-pull-requests/'

// Notification settings for "master" and "branch/pr"
def notifyMaster = [notifyAdmins: true, recipients: [culprits(), requestor()]]
def notifyBranch = [recipients: [brokenTestsSuspects(), requestor()]]

pipeline {
    agent {
        node {
            label 'alpine:jdk8-mvn-3.5'
            customWorkspace workspace().getUniqueWorkspacePath()
        }
    }
    parameters {
        booleanParam(defaultValue: true, description: 'Send email notification?', name: 'NOTIFY_EMAIL')
    }
    environment {
        // Use Pipeline Utility Steps plugin to read information from pom.xml into env variables
        ARTIFACT_ID = readMavenPom().getArtifactId()
        VERSION = readMavenPom().getVersion()
    }
    options {
        timeout(time: 2, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    stages {
        stage('Node')
        {
            steps {
                nodeInfo("mvn")
            }
        }
        stage('Building')
        {
            steps {
                withMavenPlus(timestamps: true, mavenLocalRepo: workspace().getM2LocalRepoPath(), mavenSettingsConfig: '67aaee2b-ca74-4ae1-8eb9-c8f16eb5e534')
                {
                    sh "mvn -U clean install -Dprepare.revision"
                }
            }
        }
        stage('Deploy') {
            when {
                expression {
                    (currentBuild.result == null || currentBuild.result == 'SUCCESS') &&
                    (
                        BRANCH_NAME == 'master' || (env.VERSION.contains("PR-${env.CHANGE_ID}") || env.VERSION.contains(BRANCH_NAME))
                    )
                }
            }
            steps {
                script {
                    withMavenPlus(mavenLocalRepo: workspace().getM2LocalRepoPath(), mavenSettingsConfig: 'a5452263-40e5-4d71-a5aa-4fc94a0e6833')
                    {
                        def SERVER_URL = DEPLOY_SERVER_URL;

                        if (BRANCH_NAME == 'master')
                        {
                            echo "Deploying master"

                            sh "mvn deploy" +
                               " -DskipTests" +
                               " -DaltDeploymentRepository=${SERVER_ID}::default::${SERVER_URL}"
                        }
                        else
                        {
                            echo "Deploying branch/PR"

                            SERVER_URL = PR_SERVER_URL;

                            sh "mvn deploy" +
                               " -DskipTests" +
                               " -DaltDeploymentRepository=${SERVER_ID}::default::${SERVER_URL}"
                        }
                    }
                }
            }
        }
        stage('Deploying to GitHub') {
            when {
                expression { BRANCH_NAME == 'master' && (currentBuild.result == null || currentBuild.result == 'SUCCESS') }
            }
            steps {
                withMaven(maven: 'maven-3.3.9',
                          mavenSettingsConfig: 'a5452263-40e5-4d71-a5aa-4fc94a0e6833')
                {
                    sh "mvn package -Pdeploy-release-artifact-to-github -Dmaven.test.skip=true"
                }
            }
        }
    }
    post {
        success {
            script {
                if(BRANCH_NAME == 'master' && params.TRIGGER_OS_BUILD) {
                    build job: "strongbox/strongbox", wait: false, parameters: [[$class: 'StringParameterValue', name: 'REVISION', value: '*/master']]
                }
            }
        }
        failure {
            script {
                if(params.NOTIFY_EMAIL) {
                    notifyFailed((BRANCH_NAME == "master") ? notifyMaster : notifyBranch)
                }
            }
        }
        unstable {
            script {
                if(params.NOTIFY_EMAIL) {
                    notifyUnstable((BRANCH_NAME == "master") ? notifyMaster : notifyBranch)
                }
            }
        }
        fixed {
            script {
                if(params.NOTIFY_EMAIL) {
                    notifyFixed((BRANCH_NAME == "master") ? notifyMaster : notifyBranch)
                }
            }
        }
        cleanup {
            script {
                workspace().clean()
            }
        }
    }
}
