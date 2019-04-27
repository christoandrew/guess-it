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
                step {
                    // Checkout code from repository and update any submodules
                    checkout scm
                    sh 'git submodule update --init'
                }
            }
        }

        stage('Stage Build') {
            steps {
                step {
                    //branch name from Jenkins environment variables
                    // echo "My branch is: ${env.BRANCH_NAME}"
                    script {
                        echo "Building flavor ${flavor}"
                    }
                }

                step {
                    script {
                        //build your gradle flavor, passes the current build number as a parameter to gradle
                        sh "./gradlew clean assembleDebug -PBUILD_NUMBER=${date}-${suiteRunId}"
                    }
                }
            }
        }

        stage('Stage Archive') {
            //tell Jenkins to archive the apks
            steps {
                step {
                    archiveArtifacts artifacts: 'app/build/outputs/apk/*.apk', fingerprint: true
                }
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