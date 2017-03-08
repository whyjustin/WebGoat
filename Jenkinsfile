@Library('zion-pipeline-library')
import com.sonatype.jenkins.pipeline.GitHub
import com.sonatype.jenkins.pipeline.OsTools

node {
  def gitHub

  stage('Preparation') {
    checkout scm
    sh 'git rev-parse HEAD > .git/commit-id'
    commitId = readFile('.git/commit-id')

    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: 'integrations-github-api',
                      usernameVariable: 'GITHUB_API_USERNAME', passwordVariable: 'GITHUB_API_PASSWORD']]) {
      gitHub = new GitHub(this, 'whyjustin/WebGoat', env.GITHUB_API_PASSWORD)
    }
  }
  stage('Build') {
    gitHub.gitHubStatusUpdate 'pending', 'build', 'Build in running'

    withMaven(jdk: 'JDK7', maven: 'M3', mavenSettingsConfig: 'private-settings.xml') {
      OsTools.runSafe this, 'mvn clean package'
    }

    if (currentBuild.result == 'FAILURE') {
      gitHub.gitHubStatusUpdate 'failure', 'build', 'Build failed'
      return
    } else {
      gitHub.gitHubStatusUpdate 'success', 'build', 'Build succeeded'
    }
  }
  stage('Nexus Lifecycle Analysis') {
    gitHubStatusUpdate 'pending', 'analysis', 'Nexus Lifecycle Analysis in running'

    def evaluation = nexusPolicyEvaluation failBuildOnNetworkError: false, iqApplication: 'webgoat', iqStage: 'build', jobCredentialsId: ''

    if (currentBuild.result == 'FAILURE') {
      gitHub.gitHubStatusUpdate 'failure', 'analysis', 'Nexus Lifecycle Analysis failed', "${evaluation.applicationCompositionReportUrl}"
      return
    } else {
      gitHub.gitHubStatusUpdate 'success', 'analysis', 'Nexus Lifecycle Analysis passed', "${evaluation.applicationCompositionReportUrl}"
    }
  }
  stage('Results') {
    junit '**/target/surefire-reports/TEST-*.xml'
    archive 'target/*.jar'
  }
}
