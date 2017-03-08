import groovy.json.JsonOutput

node {
  def commitId

  def runOsSafe = { command ->
    if (isUnix()) {
      sh command
    } else {
      bat command
    }
  }

  def gitHubPost = { payload, endpoint ->
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'integrations-github-api',
                      usernameVariable: 'GITHUB_API_USERNAME', passwordVariable: 'GITHUB_API_PASSWORD']]) {
      runOsSafe "curl -H \"Authorization: token ${env.GITHUB_API_PASSWORD}\" --request POST --data '${payload}' https://api.github.com/repos/whyjustin/WebGoat/${endpoint} > /dev/null"
    }
  }

  stage('Preparation') {
    checkout scm
    sh 'git rev-parse HEAD > .git/commit-id'
    commitId = readFile('.git/commit-id')
  }
  stage('Build') {
    def buildPayload = JsonOutput.toJson(
      state: 'pending',
      context: 'build',
      description: 'Build in running'
    )
    gitHubPost buildPayload, "statuses/${commitId}"

    withMaven(jdk: 'JDK7', maven: 'M3', mavenSettingsConfig: 'private-settings.xml') {
      runOsSafe "mvn clean package"
    }

    if (currentBuild.result == 'FAILURE') {
      buildPayload = JsonOutput.toJson(
        state: 'failure',
        context: 'build',
        description: 'Build failed'
      )
      gitHubPost buildPayload, "statuses/${commitId}"
      return
    } else {
      buildPayload = JsonOutput.toJson(
        state: 'success',
        context: 'build',
        description: 'Build succeeded'
      )
      gitHubPost buildPayload, "statuses/${commitId}"
    }
  }
  stage('Nexus Lifecycle Analysis') {
    def analysisPayload = JsonOutput.toJson(
      state: 'pending',
      context: 'analysis',
      description: 'Nexus Lifecycle Analysis in running'
    )
    gitHubPost analysisPayload, "statuses/${commitId}"

    def evaluation = nexusPolicyEvaluation failBuildOnNetworkError: false, iqApplication: 'webgoat', iqStage: 'build', jobCredentialsId: ''

    if (currentBuild.result == 'FAILURE') {
      analysisPayload = JsonOutput.toJson(
        state: 'failure',
        context: 'analysis',
        description: 'Nexus Lifecycle Analysis failed',
        target_url: "${evaluation.applicationCompositionReportUrl}"
      )
      gitHubPost analysisPayload, "statuses/${commitId}"
      return
    } else {
      analysisPayload = JsonOutput.toJson(
        state: 'success',
        context: 'analysis',
        description: 'Nexus Lifecycle Analysis passed',
        target_url: "${evaluation.applicationCompositionReportUrl}"
      )
      gitHubPost analysisPayload, "statuses/${commitId}"
    }
  }
  stage('Results') {
    junit '**/target/surefire-reports/TEST-*.xml'
    archive 'target/*.jar'
  }
}
