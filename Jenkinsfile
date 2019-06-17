@Library(['getChangedPaths', 'getChangelog']) _
def slackResponse = 'UNKNOWN'

pipeline {
    agent {
        node {
            label "ubuntu"
        }
    }

    options {
        disableConcurrentBuilds()
        disableResume()
        buildDiscarder(logRotator(numToKeepStr: '50'))
        timeout(time: 10, unit: 'MINUTES')
        timestamps()
    }

    environment {
        SLACK_CHANNEL = "#backend-builds"
        SERVICE_NAME  = "naga/trading/saveload"
        TAG           = "${BRANCH_NAME}-${BUILD_NUMBER}"
        MSG_PREFIX    = "*[${SERVICE_NAME}][${TAG}]*"
        ECR_REPO_URL  = "${ECR_BASE_URL}/${SERVICE_NAME}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Start') {
            steps {
                script {
                    def changelog = getChangelog()
                    def changedPaths = (getChangedPaths().isEmpty()) ? "- No file/path changes" : getChangedPaths().join(", ")

                    currentBuild.displayName = "${TAG}"

                    slackResponse = slackSend message: "${MSG_PREFIX}\n\n*Changes:*\n$changelog\n\n*Changed files/paths:*\n$changedPaths\n\n<${env.BUILD_URL}/console|Console log>",
                                        channel: "${SLACK_CHANNEL}",
                                        color: "good",
                                        teamDomain: "${env.SLACK_TEAM_DOMAIN}",
                                        tokenCredentialId: "${env.SLACK_BOT_CREDENTIALS}",
                                        botUser: true
                }
            }
        }

        stage('Build and push') {
            steps {
                sh 'rm  ~/.dockercfg || true'
                sh 'rm ~/.docker/config.json || true'

                withDockerRegistry([credentialsId:"${ECR_CREDENTIALS}", url:"https://${ECR_BASE_URL}"]) {
                    sh 'docker build -t ${ECR_REPO_URL}:${TAG} --build-arg NPM_TOKEN=${NPM_TOKEN} .'
                    sh 'docker push ${ECR_REPO_URL}:${TAG}'
                }
            }
        }

        stage('Vulnerability scan') {
            steps {
                aquaMicroscanner imageName: "${ECR_REPO_URL}:${TAG}", notCompliesCmd: 'exit 1', onDisallowed: 'ignore', outputFormat: 'html'
            }
        }
    }
    post {
        success {
            sh("git config user.name ${env.GIT_PUSH_USER}")
            sh("git config user.email ${env.GIT_PUSH_EMAIL}")
            sh("git tag -f -a ${TAG} -m '${TAG}'")
            sshagent(["${env.GIT_SSH_CREDENTIALS}"]) {
                sh("git push origin ${TAG}")
            }
            script {
                slackSend message: "${MSG_PREFIX}\n\nPipeline finished after ${currentBuild.durationString.replace(' and counting', '')}\n\n<${env.BUILD_URL}|Build log>",
                    color: "good",
                    channel: "${slackResponse.threadId}",
                    teamDomain: "${env.SLACK_TEAM_DOMAIN}",
                    tokenCredentialId: "${env.SLACK_BOT_CREDENTIALS}",
                    botUser: true
            }
            sh """
            curl -XPOST 'http://release-tool-api.devenv/deploy' -H 'Content-Type: application/json' -d'{\"tag\":\"${TAG}\",\"service-name\": \"admin-service-api\"}'
            """
        }
        failure {
            script {
                slackSend message: "${MSG_PREFIX}\n\nPipeline failed after ${currentBuild.durationString.replace(' and counting', '')}\n<${env.BUILD_URL}|Build log>",
                    color: "danger",
                    channel: "${slackResponse.threadId}",
                    replyBroadcast: true,
                    teamDomain: "${env.SLACK_TEAM_DOMAIN}",
                    tokenCredentialId: "${env.SLACK_BOT_CREDENTIALS}",
                    botUser: true
            }
        }
    }
}