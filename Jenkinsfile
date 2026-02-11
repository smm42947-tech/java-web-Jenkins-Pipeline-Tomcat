pipeline {
  agent any

  tools {
    jdk 'JDK17'
    maven 'M3'
  }

  environment {
    SONAR_HOST_URL = 'http://sonarqube:9000'
    NEXUS_BASE_URL = 'http://nexus:8081'
    TOMCAT_MANAGER = 'http://tomcat:8080/manager/text'
    APP_CONTEXT = '/demo'
  }

  options {
    disableConcurrentBuilds()
    timestamps()
  }

  stages {
    stage('Checkout') { steps { checkout scm } }

    stage('Prepare Maven Settings (Nexus)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'nexus-creds',
                                          usernameVariable: 'NEXUS_USER',
                                          passwordVariable: 'NEXUS_PASS')]) {
          writeFile file: 'ci-settings.xml', text: """
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              https://maven.apache.org/xsd/settings-1.0.0.xsd">
  <servers>
    <server>
      <id>nexus-releases</id>
      <username>${NEXUS_USER}</username>
      <password>${NEXUS_PASS}</password>
    </server>
    <server>
      <id>nexus-snapshots</id>
      <username>${NEXUS_USER}</username>
      <password>${NEXUS_PASS}</password>
    </server>
  </servers>
</settings>
""".stripIndent()
        }
      }
    }

    stage('Build & Test') {
      steps { sh 'mvn -s ci-settings.xml -B -U clean package' }
      post {
        always {
          junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
          archiveArtifacts artifacts: 'target/*.war', fingerprint: true
        }
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
          sh """
            mvn -s ci-settings.xml sonar:sonar \
              -Dsonar.host.url=${SONAR_HOST_URL} \
              -Dsonar.login=${SONAR_TOKEN} \
              -Dsonar.projectKey=java-web-jenkins-pipeline-tomcat
          """
        }
      }
    }

    stage('Publish to Nexus') {
      steps { sh 'mvn -s ci-settings.xml -B deploy' }
    }

    stage('Deploy to Tomcat') {
      steps {
        script {
          def war = sh(returnStdout: true, script: "ls -1 target/*.war | head -n 1").trim()
          if (!war) { error "No WAR found in target/" }
          withCredentials([usernamePassword(credentialsId: 'tomcat-manager-creds',
                                            usernameVariable: 'TOMCAT_USER',
                                            passwordVariable: 'TOMCAT_PASS')]) {
            sh """
              curl -sf -u ${TOMCAT_USER}:${TOMCAT_PASS} "${TOMCAT_MANAGER}/undeploy?path=${APP_CONTEXT}" || true
              curl -sf -u ${TOMCAT_USER}:${TOMCAT_PASS} -T "${war}" \\
                   "${TOMCAT_MANAGER}/deploy?path=${APP_CONTEXT}&update=true"
            """
          }
        }
      }
    }
  }

  post {
    success {
      echo "Deployed: http://localhost:8083${APP_CONTEXT}"
    }
  }
}
