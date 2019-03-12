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
    options {
        timeout(time: 1, unit: 'HOURS')
        disableConcurrentBuilds()
    }
    stages {
        stage('Node')
        {
            steps {
                nodeInfo("mvn")
            }
        }
        stage('Build')
        {
            steps {
                withMavenPlus(mavenLocalRepo: workspace().getM2LocalRepoPath(), mavenSettingsConfig: 'a5452263-40e5-4d71-a5aa-4fc94a0e6833')
                {
                    sh "mvn -U clean install -Dmaven.test.failure.ignore=true"
                }
            }
        }
        stage('Deploy') {
            when {
                expression { (currentBuild.result == null || currentBuild.result == 'SUCCESS') }
            }
            steps {
                script {
                    withMavenPlus(mavenLocalRepo: workspace().getM2LocalRepoPath(), mavenSettingsConfig: 'a5452263-40e5-4d71-a5aa-4fc94a0e6833')
                    {

                        def SERVER_URL;

                        if (BRANCH_NAME == 'master') {
                            echo "Deploying master..."
                            SERVER_URL = DEPLOY_SERVER_URL;

                            // We are temporarily reverting back.
                            sh "mvn deploy" +
                               " -DskipTests" +
                               " -DaltDeploymentRepository=${SERVER_ID}::default::${DEPLOY_SERVER_URL}"

                            def APPROVE_RELEASE=false
                            def APPROVED_BY=""

                            try {
                                timeout(time: 15, unit: 'MINUTES')
                                {
                                    rocketSend attachments: [[
                                         authorIcon: 'https://jenkins.carlspring.org/static/fd850815/images/headshot.png',
                                         authorName: 'Jenkins',
                                         color: '#f4bc0d',
                                         text: 'Job is pending release approval! If no action is taken within 15 minutes, it will abort releasing.',
                                         title: env.JOB_NAME + ' #' + env.BUILD_NUMBER,
                                         titleLink: env.BUILD_URL
                                    ]], message: '', rawMessage: true, channel: '#strongbox-devs'

                                    APPROVE_RELEASE = input message: 'Do you want to release and deploy this version?',
                                                            submitter: 'administrators,strongbox,strongbox-pro'
                                }
                            }
                            catch(err)
                            {
                                APPROVE_RELEASE = false
                            }

                            if(APPROVE_RELEASE == true || APPROVE_RELEASE.equals(null))
                            {
                                echo "Preparing GPG keys..."
                                sh "gpg --list-secret-keys"
                                sh "gpg --import ${8E3885CCF99DE3E178C55F32CEE144B17ECAC99A}"
                                sh "gpg --list-secret-keys"

                                echo "Set upstream branch..."
                                sh "git branch --set-upstream-to=origin/master master"

                                echo "Preparing release and tag..."
                                sh "mvn -B release:clean release:prepare"

                                def releaseProperties = readProperties(file: "release.properties");
                                def RELEASE_VERSION = releaseProperties["scm.tag"]

                                echo "Deploying " + RELEASE_VERSION

                                sh "mvn -B release:perform -DserverId=${SERVER_ID} -DdeployUrl=${RELEASE_SERVER_URL}"
                            }
                            else
                            {
                                echo "Deployment has been skipped, because it was not approved."
                            }

                        } else {
                            echo "Deploying branch/PR"

                            def pom = readMavenPom file: 'pom.xml'
	                        def VERSION_ID = pom.version

                          	def gitBranch = sh(returnStdout: true, script: 'git rev-parse --abbrev-ref HEAD').trim()

                            SERVER_URL = PR_SERVER_URL;
                            VERSION_ID = VERSION_ID.replaceAll("-SNAPSHOT", "") + "-${GIT_BRANCH}";

                            sh "mvn versions:set -DnewVersion=${VERSION_ID}-SNAPSHOT"
                            sh "mvn versions:commit"
                        }

                        sh "mvn package deploy:deploy" +
                           " -Drat.ignoreErrors=true" +
                           " -Dmaven.test.skip=true" +
                           " -DaltDeploymentRepository=${SERVER_ID}::default::${SERVER_URL}"
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
        always {
            // (fallback) record test results even if withMaven should have done that already.
            junit '**/target/*-reports/*.xml'
        }
        cleanup {
            script {
                workspace().clean()
            }
        }
    }
}
