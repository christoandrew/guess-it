#!groovy
pipeline {
    environment {
        date = new Date().format('yyyyMMdd')
        suiteRunId = UUID.randomUUID().toString()
    }

    agent any

    parameters {

        gitParameter(name: 'PULL_REQUESTS',
                type: 'PT_PULL_REQUEST',
                defaultValue: '1',
                description: 'SELECT PR TO BUILD',
                sortMode: 'DESCENDING_SMART')

        choice(name: "BUILD_TYPE",
                description: "SELECT BUILD TYPE",
                choices: "debug\nrelease\ntrainDebug\ntrainRelease\n")
    }

    stages {

        stage('Initializing Build') {
            steps {
                script {
                    getPullRequest()
                    echo sh(returnStdout: true, script: 'env')
                }
            }
        }

        stage('Stage Checkout') {
            steps {
                // Checkout code from repository and update any submodules

                script {

                    def buildRef = env.pullRequest != null ? "pr/${env.pullRequest}/head" : env.branchName

                    echo "Running build with ${buildRef}"

                    withCredentials([usernamePassword(credentialsId: '3452d8b5-7876-4e02-84f3-53b512a2dc83',
                            usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {

                        checkout([$class                           : 'GitSCM',
                                  branches                         : [[name: "${buildRef}"]],
                                  doGenerateSubmoduleConfigurations: false,
                                  extensions                       : [],
                                  submoduleCfg                     : [],
                                  userRemoteConfigs                : [[credentialsId: '3452d8b5-7876-4e02-84f3-53b512a2dc83',
                                                                       refspec      : '+refs/pull/*:refs/remotes/origin/pr/* +refs/heads/*:refs/remotes/origin/*',
                                                                       url          : 'https://github.com/FenixIntl/FenixGoNative.git']]])


                        sh 'git submodule update --init'

                    }
                }
            }
        }

        stage('Stage Build') {
            steps {
                script {
                    def flavor = buildFlavor(params.BUILD_TYPE)
                    //build your gradle flavor, passes the current build number as a parameter to gradle
                    sh "PR=${pullRequest} BUILD=${BUILD_NUMBER} DATE=${date} ./gradlew clean ${flavor} -PBUILD_NUMBER=${suiteRunId}"
                }

            }
        }

        stage("Stage Test") {
            steps {
                //sh './gradlew check'
                print("Stage Test")
            }
        }

        stage('Stage Archive') {
            steps {
                archiveArtifacts artifacts: 'app/build/outputs/apk/**/*.apk', fingerprint: true
            }
        }

//        stage('Stage Email Archive') {
//            steps {
//                emailext attachLog: true,
//                        //attachmentsPattern: 'app/build/outputs/apk/debug/*.apk',
//                        body: "${date}-${suiteRunId}",
//                        compressLog: true,
//                        replyTo: 'jenkins-ci@ci-server.com',
//                        subject: 'Build ',
//                        to: 'andrew.christoandrew.christo@gmail.com'
//                script {
//                    emailext attachLog: true, body: '${BUILD_STATUS}', compressLog: true,
//                            recipientProviders: [culprits(), developers()],
//                            subject: 'Build ${BUILD_NUMBER}', to: 'awekesa@fenixintl.com'
//                }
//            }
//        }

//        stage('Deploy') {
//            steps {
//                echo "Deploying"
//            }
//        }
    }

    //stage 'Stage Upload To Fabric'
    //sh "./gradlew crashlyticsUploadDistribution${flavor}Debug  -PBUILD_NUMBER=${env.BUILD_NUMBER}"
}

def getPullRequest() {
    if (isParameterizedBuild() && !isPullRequestBuild() && !isBranchBuild()) {
        env.pullRequest = params.PULL_REQUESTS
    } else if (isPullRequestBuild()) {
        env.branchName = env.GITHUB_PR_SOURCE_BRANCH
    } else {
        env.branchName = env.GITHUB_BRANCH_NAME
    }
}

def isParameterizedBuild() {
    return (env.PULL_REQUESTS != null) && (env.BUILD_TYPE != null)
}

def isPullRequestBuild() {
    return env.GITHUB_PR_NUMBER != null
}

def isBranchBuild() {
    return env.GITHUB_BRANCH_NAME != null
}


@NonCPS
def buildFlavor(buildType) {
    String flavor = "assembleDebug"
    switch (buildType) {
        case "debug":
            flavor = "assembleDebug"
            break
        case "release":
            flavor = "assembleRelease"
            break
        case "trainRelease":
            flavor = "assembleTrainRelease"
            break
        case "trainDebug":
            flavor = "assembleTrainDebug"
            break
        default:
            break
    }

    return flavor.toString()
}
