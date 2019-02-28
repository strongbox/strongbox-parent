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
        GROUP_ID = readMavenPom().getGroupId()
        ARTIFACT_ID = readMavenPom().getArtifactId()
        VERSION = readMavenPom().getVersion()
    }
    options {
        timeout(time: 120, unit: 'MINUTES')
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
                        BRANCH_NAME == 'master' ||
                        env.VERSION.contains("PR-${env.CHANGE_ID}") ||
                        env.VERSION.contains(BRANCH_NAME)
                    )
                }
            }
            steps {
                script {
                    withMavenPlus(mavenLocalRepo: workspace().getM2LocalRepoPath(), mavenSettingsConfig: 'a5452263-40e5-4d71-a5aa-4fc94a0e6833', options: [artifactsPublisher(), mavenLinkerPublisher()], tempBinDir: '') {
                        echo "Deploying " + GROUP_ID + ":" + ARTIFACT_ID + ":" + VERSION

                        if (BRANCH_NAME == 'master')
                        {
                            // We are temporarily reverting back.
                            sh "mvn deploy" +
                               " -DskipTests" +
                               " -DaltDeploymentRepository=${SERVER_ID}::default::${SNAPSHOT_SERVER_URL}"

                            // This will be used in the future.
                            // def APPROVE_RELEASE=false
                            // def APPROVED_BY=""

                            //try {
                            //    timeout(time: 115, unit: 'MINUTES')
                            //    {
                            //        rocketSend attachments: [[
                            //             authorIcon: 'https://jenkins.carlspring.org/static/fd850815/images/headshot.png',
                            //             authorName: 'Jenkins',
                            //             color: '#f4bc0d',
                            //             text: 'Job is pending release approval! If no action is taken within an hour, it will abort releasing.',
                            //             title: env.JOB_NAME + ' #' + env.BUILD_NUMBER,
                            //             titleLink: env.BUILD_URL
                            //        ]], message: '', rawMessage: true, channel: '#strongbox-devs'
                            //
                            //        APPROVE_RELEASE = input message: 'Do you want to release and deploy this version?',
                            //                                submitter: 'administrators,strongbox,strongbox-pro'
                            //    }
                            //}
                            //catch(err)
                            //{
                            //    APPROVE_RELEASE = false
                            //}

                            //if(APPROVE_RELEASE == true || APPROVE_RELEASE.equals(null))
                            //{
                            //    echo "Set upstream branch..."
                            //    sh "git branch --set-upstream-to=origin/master master"

                            //    echo "Preparing release and tag..."
                            //    sh "mvn -B release:clean release:prepare"

                            //    def releaseProperties = readProperties(file: "release.properties");
                            //    def RELEASE_VERSION = releaseProperties["scm.tag"]

                            //    echo "Deploying " + RELEASE_VERSION

                            //    sh "mvn -B release:perform -DserverId=${SERVER_ID} -DdeployUrl=${RELEASE_SERVER_URL}"
                            //}
                            //else
                            //{
                            //    echo "Deployment has been skipped, because it was not approved."
                            //}
                        }
                        else
                        {
                            sh "mvn deploy" +
                               " -DskipTests" +
                               " -DaltDeploymentRepository=${SERVER_ID}::default::${PR_SERVER_URL}"
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
        cleanup {
            script {
                workspace().clean()
            }
        }
    }
}
