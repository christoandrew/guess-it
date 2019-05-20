#!groovy​
pipeline {
    environment {
        date = new Date().format('yyyyMMdd')
        suiteRunId = UUID.randomUUID().toString()
        flavor = "master"
    }
    
    agent any
    
    parameters {
        gitParameter name: 'BRANCH',
                     type: 'PT_BRANCH',
                     defaultValue: 'master'
    }
    
    stages {
        // Mark the code checkout 'stage'....
        stage('Stage Checkout') {
            steps {
                // Checkout code from repository and update any submodules
                checkout([$class: 'GitSCM',
                          branches: [[name: "${BRANCH}"]],
                          doGenerateSubmoduleConfigurations: false,
                          extensions: [],
                          gitTool: 'Default',
                          submoduleCfg: [],
                          userRemoteConfigs: [[url: 'https://github.com/christoandrew/guess-it.git']]
                        ])
                //checkout scm
                sh 'git submodule update --init'
            }
        }

        stage('Stage Build') {
            steps {
                //branch name from Jenkins environment variables
                // echo "My branch is: ${env.BRANCH_NAME}"
                script {
                    echo "Building build type ${BUILD_TYPE}"
                    echo "Building flavor ${BRANCH}"
                    echo "Building flavor ${BUILD_FLAVOR}"
                }
                script {
                    //build your gradle flavor, passes the current build number as a parameter to gradle
                    sh "./gradlew clean "+ buildFlavor("${BUILD_TYPE}") + "-PBUILD_NUMBER=${date}-${suiteRunId}"
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
                input message: "Deploy?"
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
def buildFlavor(buidlType){
    String flavor = "assembleDebug"
    switch(buildType){
      case "debug":
        flavor = "assembleDebug"
        break
      case "release":
        flavor = "assembleRelease"
        break
      default:
        break
    }
    
    return flavor.toString()
}
