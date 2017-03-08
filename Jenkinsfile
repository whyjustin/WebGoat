import groovy.json.JsonOutput

def runOsSafe(GString command) {
  if (isUnix()) {
    sh command
  } else {
    bat command
  }
}

node {
  def javaHome
  def mvnHome
  def commitId
  def gitHubApiToken

  def gitHubPost = { payload, endpoint ->
    runOsSafe "curl -H \"Authorization: token ${gitHubApiToken}\" --request POST --data '${payload}' https://api.github.com/repos/whyjustin/WebGoat/${endpoint} > /dev/null"
  }

  stage('Preparation') {
    checkout scm
    sh 'git rev-parse HEAD > .git/commit-id'
    commitId = readFile('.git/commit-id')

    javaHome = tool 'Java 7'
    mvnHome = tool 'M3'
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'integrations-github-api',
                      usernameVariable: 'GITHUB_API_USERNAME', passwordVariable: 'GITHUB_API_PASSWORD']]) {
      gitHubApiToken = env.GITHUB_API_PASSWORD
    }
  }
  stage('Build') {
    def buildPayload = JsonOutput.toJson(
      state: 'pending',
      context: 'build',
      description: 'Build in running'
    )
    gitHubPost buildPayload, "statuses/${commitId}".toString()

    runOsSafe "JAVA_HOME=${javaHome} ${mvnHome}/bin/mvn clean package"

    if (currentBuild.result == 'FAILURE') {
      buildPayload = JsonOutput.toJson(
        state: 'failure',
        context: 'build',
        description: 'Build failed'
      )
      gitHubPost buildPayload, "statuses/${commitId}".toString()
      return
    } else {
      buildPayload = JsonOutput.toJson(
        state: 'success',
        context: 'build',
        description: 'Build succeeded'
      )
      gitHubPost buildPayload, "statuses/${commitId}".toString()
    }
  }
  stage('Nexus Lifecycle Analysis') {
    def analysisPayload = JsonOutput.toJson(
      state: 'pending',
      context: 'analysis',
      description: 'Nexus Lifecycle Analysis in running'
    )
    gitHubPost analysisPayload, "statuses/${commitId}".toString()

    def evaluation = nexusPolicyEvaluation failBuildOnNetworkError: false, iqApplication: 'webgoat', iqStage: 'build', jobCredentialsId: ''

    if (currentBuild.result == 'FAILURE') {
      analysisPayload = JsonOutput.toJson(
        state: 'failure',
        context: 'analysis',
        description: 'Nexus Lifecycle Analysis failed',
        target_url: "${evaluation.applicationCompositionReportUrl}"
      )
      gitHubPost analysisPayload, "statuses/${commitId}".toString()
      return
    } else {
      analysisPayload = JsonOutput.toJson(
        state: 'success',
        context: 'analysis',
        description: 'Nexus Lifecycle Analysis passed',
        target_url: "${evaluation.applicationCompositionReportUrl}"
      )
      gitHubPost analysisPayload, "statuses/${commitId}".toString()
    }
  }
  stage('Results') {
    junit '**/target/surefire-reports/TEST-*.xml'
    archive 'target/*.jar'
  }
}
