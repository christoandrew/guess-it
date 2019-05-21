#!groovy​
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
        // Mark the code checkout 'stage'....
        stage('Stage Checkout') {
            steps {
                // Checkout code from repository and update any submodules

                script{
                    checkout([$class                           : 'GitSCM',
                              branches                         : [[name: "pr/${params.PULL_REQUESTS}/head"]],
                              doGenerateSubmoduleConfigurations: false,
                              extensions                       : [],
                              gitTool                          : 'Default',
                              submoduleCfg                     : [],
                              userRemoteConfigs                : [[refspec: '+refs/pull/*:refs/remotes/origin/pr/*', url: 'https://github.com/christoandrew/guess-it.git']]])

                    sh 'git submodule update --init'
                }
            }
        }

        stage('Stage Build') {
            steps {
                //branch name from Jenkins environment variables
                // echo "My branch is: ${env.BRANCH_NAME}"
                script {
                    def flavor = buildFlavor(params.BUILD_TYPE)
                    def pr = "pr/${params.PULL_REQUESTS}/head"
                    //build your gradle flavor, passes the current build number as a parameter to gradle
                    sh "PR=${pr} BUILD=${BUILD_NUMBER} DATE=${date} ./gradlew clean ${flavor} -PBUILD_NUMBER=${suiteRunId}"
                }

            }
        }

        stage("Stage Test") {
            steps {
                sh './gradlew check'
            }
        }

        stage('Stage Archive') {
            //tell Jenkins to archive the apks

            steps {
                archiveArtifacts artifacts: 'app/build/outputs/apk/**/*.apk', fingerprint: true
            }
        }

        stage('Stage Email Archive') {
            steps {
                emailext attachLog: true,
                        //attachmentsPattern: 'app/build/outputs/apk/debug/*.apk',
                        body: "${date}-${suiteRunId}",
                        compressLog: true,
                        replyTo: 'jenkins-ci@ci-server.com',
                        subject: 'Build ',
                        to: 'andrew.christoandrew.christo@gmail.com'
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying"
            }
        }
    }

    //stage 'Stage Upload To Fabric'
    //sh "./gradlew crashlyticsUploadDistribution${flavor}Debug  -PBUILD_NUMBER=${env.BUILD_NUMBER}"
}

// Pulls the android flavor out of the branch name the branch is prepended with /QA_
@NonCPS
def flavor(branchName) {
    def matcher = (env.BRANCH_NAME =~ /MOB_([a-z_]+)/)
    assert matcher.matches()
    matcher[0][1]
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
