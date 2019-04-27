pipeline {
    environment {
        date = new Date().format('yyyyMMdd')
        suiteRunId = UUID.randomUUID().toString()
        flavor = "master"
    }
    agent any
    stages {
        // Mark the code checkout 'stage'....
        stage('Stage Checkout') {
            steps {
                // Checkout code from repository and update any submodules
                checkout scm
                sh 'git submodule update --init'
            }
        }

        stage('Stage Build') {
            steps {
                //branch name from Jenkins environment variables
                // echo "My branch is: ${env.BRANCH_NAME}"
                script {
                    echo "Building flavor ${flavor}"
                }
                script {
                    //build your gradle flavor, passes the current build number as a parameter to gradle
                    sh "./gradlew clean assembleDebug -PBUILD_NUMBER=${date}-${suiteRunId}"
                }

            }
        }

        stage('Stage Archive') {
            //tell Jenkins to archive the apks

            steps {
                archiveArtifacts artifacts: 'app/build/outputs/apk/**/*.apk', fingerprint: true
            }
        }

        stage('Email Archive') {
            emailext attachLog: true,
                    attachmentsPattern: 'app/build/outputs/apk/debug/*.apk',
                    body: '${date}-${suiteRunId}',
                    compressLog: true,
                    replyTo: 'jenkins-ci@ci-server.com',
                    subject: 'Build ',
                    to: 'andrew.christoandrew.christo@gmail.com'
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