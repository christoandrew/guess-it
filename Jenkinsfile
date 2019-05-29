#!groovy

pipeline {
    environment {
        date = new Date().format('yyyyMMdd')
        suiteRunId = UUID.randomUUID().toString()
        //branchName = "master"
    }

    agent any

    parameters {

        gitParameter(name: 'BRANCH',
                type: 'PT_BRANCH',
                defaultValue: '1',
                description: 'SELECT BRANCH TO BUILD',
                sortMode: 'DESCENDING_SMART')

        choice(name: "BUILD_TYPE",
                description: "SELECT BUILD TYPE",
                choices: "Debug\nRelease\nTrainDebug\nTrainRelease\n")
    }

    stages {

        stage('Initializing Build') {
            steps {
                script {
                    getPullRequest()
                    def buildType = params.BUILD_TYPE != null ? params.BUILD_TYPE : "Debug"
                    env.buildType = buildType
                    echo sh(returnStdout: true, script: 'env')
                }
            }
        }

        stage('Stage Checkout') {
            steps {
                // Checkout code from repository and update any submodules
                script {

                    def buildRef = env.branchName

                    echo "Running build with ${buildRef}"

                    withCredentials([usernamePassword(credentialsId: '557caf57-caee-4f33-88c5-36e06a8150cb',
                            usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        // Checkout from SCM
                        checkout([$class                           : 'GitSCM',
                                  branches                         : [[name: "${buildRef}"]],
                                  doGenerateSubmoduleConfigurations: false,
                                  extensions                       : [],
                                  submoduleCfg                     : [],
                                  userRemoteConfigs                : [
                                          [credentialsId: '3452d8b5-7876-4e02-84f3-53b512a2dc83',
                                           refspec      : '+refs/pull/*:refs/remotes/origin/pr/* +refs/heads/*:refs/remotes/origin/*',
                                           url          : 'https://github.com/christoandrew/guess-it.git']
                                  ]
                        ])

                        sh 'git submodule update --init'

                    }
                }
            }
        }

        stage('Stage Run Lint') {
            steps {
                script {
                    // Run gradle lint
                    sh "./gradlew lint"

                    // Archive lint result
                    archiveArtifacts artifacts: 'app/build/reports/lint-results.html', fingerprint: true
                    archiveArtifacts artifacts: 'app/build/reports/lint-results.xml', fingerprint: true
                    // Publish Lint Results
                    publishLintResults()
                    // Run lint generate jenkins report
                    androidLint canRunOnFailed: true, defaultEncoding: 'UTF-8', healthy: '',
                            shouldDetectModules: true,
                            unHealthy: '', useDeltaValues: true,
                            usePreviousBuildAsReference: true,
                            useStableBuildAsReference: true
                }
            }
        }

        stage("Stage Testing") {
            steps {
                script {
                    // Run unit tests for build type
                    sh "./gradlew test${buildType}UnitTest --stacktrace"
                    // Publish Test Report
                    publishTestReport()
                }
            }
        }

        stage('Stage Build') {
            steps {
                script {
                    def flavor = buildFlavor(params.BUILD_TYPE)
                    //build your gradle flavor, passes the current build number as a parameter to gradle
                    sh "./gradlew clean"
                    sh "./gradlew ${flavor}"
                    setGithubPRStatus()
                }

            }
        }

        stage('Stage Archive APK') {
            steps {
                archiveArtifacts artifacts: 'app/build/outputs/apk/**/*.apk', fingerprint: true
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying"
            }
        }

        stage('Stage Send Email Notification') {
            steps {
                emailNotification()
            }
        }
    }
}

def getPullRequest() {
    if (isParameterizedBuild() && !isPullRequestBuild() && !isBranchBuild()) {
        env.branchName = params.BRANCH
    } else if (isPullRequestBuild()) {
        env.branchName = env.GITHUB_PR_SOURCE_BRANCH
    } else {
        env.branchName = env.GITHUB_BRANCH_NAME
    }
}

def isParameterizedBuild() {
    return (env.BRANCH != null) && (env.BUILD_TYPE != null)
}

def isPullRequestBuild() {
    return env.GITHUB_PR_NUMBER != null
}

def isBranchBuild() {
    return env.GITHUB_BRANCH_NAME != null
}

def publishTestReport() {

    publishHTML([allowMissing         : false,
                 alwaysLinkToLastBuild: false,
                 keepAll              : false,
                 reportDir            : "app/build/reports/tests/test${buildType}UnitTest/",
                 reportFiles          : 'index.html',
                 reportName           : "${buildType} Test Result Report ${env.branchName}-${BUILD_NUMBER}",
                 reportTitles         : ''])

}

def publishLintResults() {
    publishHTML([allowMissing         : false,
                 alwaysLinkToLastBuild: false,
                 keepAll              : false,
                 reportDir            : "app/build/reports/",
                 reportFiles          : 'lint-results.html',
                 reportName           : "${buildType} Lint Result Report ${env.branchName}-${BUILD_NUMBER}",
                 reportTitles         : ''])
}

def emailNotification() {
    def to = emailextrecipients([[$class: 'CulpritsRecipientProvider'],
                                 [$class: 'DevelopersRecipientProvider'],
                                 [$class: 'RequesterRecipientProvider']])
    String currentResult = currentBuild.currentResult
    String previousResult = currentBuild.getPreviousBuild().currentResult
    String currentBuildStatus = currentBuild.currentResult

    def causes = currentBuild.rawBuild.getCauses()
    // E.g. 'started by user', 'triggered by scm change'
    def cause = null
    if (!causes.isEmpty()) {
        cause = causes[0].getShortDescription()
    }

    // Ensure we don't keep a list of causes, or we get
    // "java.io.NotSerializableException: hudson.model.Cause$UserIdCause"
    // see http://stackoverflow.com/a/37897833/509706
    causes = null

    String subject = "$env.JOB_NAME $env.branchName $env.BUILD_NUMBER: $currentBuildStatus"

    String body = """
                    <p>
                        Build $env.BUILD_NUMBER ran on $env.NODE_NAME and terminated with $currentBuildStatus.
                    </p>
                    <p>
                        Build trigger: $cause
                    </p>
                    <p>
                        Build status: ${currentBuildStatus}
                    </p>
                    <p>
                        See: <a href="$env.BUILD_URL">$env.BUILD_URL</a>
                    </p>

                """
    String log = currentBuild.rawBuild.getLog(Integer.MAX_VALUE).join('\n')

    if (currentBuildStatus == 'SUCCESS') {
        body = body + """

                
                """
    } else {
        body = body + """
            <h2>Last lines of output</h2>
            <pre>$log</pre>
        """
    }

    if (to != null && !to.isEmpty()) {
        // Email on any failures, and on first success.
        if (currentResult != 'SUCCESS' || currentResult != previousResult) {
            mail to: to, subject: subject, body: body, mimeType: "text/html",
                    from: "jenkins-ci@citrain.com"
        }

        echo 'Sent email notification'
    }
}

def setGithubPRStatus(){
    githubPRStatusPublisher buildMessage: message(failureMsg: githubPRMessage('Can\'t set status; build failed.'),
            successMsg: githubPRMessage('Can\'t set status; build succeeded.')), errorHandler: statusOnPublisherError('UNSTABLE'),
            statusMsg: githubPRMessage('${GITHUB_PR_COND_REF} run ended'), unstableAs: 'FAILURE'
}

@NonCPS
def buildFlavor(buildType) {
    String flavor = "assembleDebug"
    switch (buildType) {
        case "Debug":
            flavor = "assembleDebug"
            break
        case "Release":
            flavor = "assembleRelease"
            break
        case "TrainRelease":
            flavor = "assembleTrainRelease"
            break
        case "TrainDebug":
            flavor = "assembleTrainDebug"
            break
        default:
            break
    }

    return flavor.toString()
}
