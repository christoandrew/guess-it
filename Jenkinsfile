node {
    // Mark the code checkout 'stage'....
    stages{
        stage('Stage Checkout'){
            steps{
                // Checkout code from repository and update any submodules
                checkout scm
                sh 'git submodule update --init'
            }
        }

        stage('Stage Build'){
            steps{
                //branch name from Jenkins environment variables
                echo "My branch is: ${env.BRANCH_NAME}"

                def flavor = "master"
                echo "Building flavor ${flavor}"

                //build your gradle flavor, passes the current build number as a parameter to gradle
                sh "./gradlew clean assemble${flavor}Debug -PBUILD_NUMBER=${env.BUILD_NUMBER}"
            }
        }

        stage('Stage Archive'){
            //tell Jenkins to archive the apks
            archiveArtifacts artifacts: 'app/build/outputs/apk/*.apk', fingerprint: true
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