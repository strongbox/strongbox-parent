@Library('jenkins-shared-libraries') _

def SERVER_ID  = 'carlspring'
def SNAPSHOT_SERVER_URL = 'https://repo.carlspring.org/content/repositories/carlspring-oss-snapshots'
def RELEASE_SERVER_URL = 'https://repo.carlspring.org/content/repositories/carlspring-oss-releases/'
def PR_SERVER_URL = 'https://repo.carlspring.org/content/repositories/carlspring-oss-pull-requests/'

// Notification settings for "master" and "branch/pr"
def notifyMaster = [notifyAdmins: true, recipients: [culprits(), requestor()]]
def notifyBranch = [recipients: [brokenTestsSuspects(), requestor()]]

pipeline {
    agent {
        label 'alpine-jdk8-mvn3.6'
    }
    parameters {
        booleanParam(defaultValue: true, description: 'Send email notification?', name: 'NOTIFY_EMAIL')
    }
    environment {
        // Use Pipeline Utility Steps plugin to read information from pom.xml into env variables
        GROUP_ID = readMavenPom().getGroupId()
        ARTIFACT_ID = readMavenPom().getArtifactId()
        VERSION = readMavenPom().getVersion()
    }
    options {
        timeout(time: 120, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    stages {
        stage('Node') {
            steps {
                container("maven") {
                    nodeInfo("mvn")
                }
            }
        }
        stage('Building') {
            steps {
                container("maven") {
                    withMavenPlus(timestamps: true, mavenLocalRepo: workspace().getM2LocalRepoPath(), mavenSettingsConfig: '67aaee2b-ca74-4ae1-8eb9-c8f16eb5e534') {
                        sh "mvn -U clean install -Dprepare.revision"
                    }
                }
            }
        }
        stage('Deploy') {
            when {
                expression {
                    (currentBuild.result == null || currentBuild.result == 'SUCCESS') &&
                    (
                        BRANCH_NAME == 'master' ||
                        env.VERSION.contains("PR-${env.CHANGE_ID}") ||
                        env.VERSION.contains(BRANCH_NAME)
                    )
                }
            }
            steps {
                script {
                    container("maven") {
                        withMavenPlus(mavenLocalRepo: workspace().getM2LocalRepoPath(), mavenSettingsConfig: 'a5452263-40e5-4d71-a5aa-4fc94a0e6833', options: [artifactsPublisher(), mavenLinkerPublisher()], tempBinDir: '') {
                            echo "Deploying " + GROUP_ID + ":" + ARTIFACT_ID + ":" + VERSION
    
                            def SERVER_URL = PR_SERVER_URL
                            if (BRANCH_NAME == 'master') {
                                SERVER_URL = SNAPSHOT_SERVER_URL
                            }

                            sh "mvn deploy -DskipTests -DaltDeploymentRepository=${SERVER_ID}::default::${SERVER_URL}"
                        }
                    }
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
    }
}
