@Library('jenkins-shared-libraries') _

def SERVER_ID  = 'carlspring'
def SNAPSHOT_SERVER_URL = 'https://eu.repo.carlspring.org/content/repositories/carlspring-oss-snapshots'
def RELEASE_SERVER_URL = 'https://eu.repo.carlspring.org/content/repositories/carlspring-oss-releases'
def PR_SERVER_URL = 'https://eu.repo.carlspring.org/content/repositories/carlspring-oss-pull-requests'

// Notification settings for "master" and "branch/pr"
def notifyMaster = [notifyAdmins: true, recipients: [culprits(), requestor()]]
def notifyBranch = [recipients: [brokenTestsSuspects(), requestor()]]
def isMasterBranch = 'master'.equals(env.BRANCH_NAME);

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
        skipStagesAfterUnstable()
    }
    triggers {
        cron(isMasterBranch ? '' : 'H * * * 5')
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
                    withMavenPlus(
                        timestamps: true,
                        mavenLocalRepo: workspace().getM2LocalRepoPath(),
                        mavenSettingsConfig: '67aaee2b-ca74-4ae1-8eb9-c8f16eb5e534',
                        publisherStrategy: 'EXPLICIT'
                    ) {
                        sh "mvn -U clean install -Dprepare.revision"
                    }
                }
            }
        }
        stage('Deploy') {
            when {
                expression {
                    isMasterBranch || isDeployableTempVersion()
                }
            }
            steps {
                script {
                    container("maven") {
                        withMavenPlus(
                            mavenLocalRepo: workspace().getM2LocalRepoPath(),
                            mavenSettingsConfig: 'a5452263-40e5-4d71-a5aa-4fc94a0e6833',
                            options: [
                                artifactsPublisher(disabled: true),
                                // https://issues.jenkins-ci.org/browse/JENKINS-46152
                                pipelineGraphPublisher(includeReleaseVersions: true, skipDownstreamTriggers: true),
                                mavenLinkerPublisher()
                            ],
                            publisherStrategy: 'EXPLICIT'
                        ) {
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
