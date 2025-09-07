pipeline {
  agent any

  // MUST match Manage Jenkins → Tools
  tools {
    jdk   'java-17'
    maven 'Maven'
  }

  environment {
    REPO_URL         = 'https://github.com/preyelg/numberguessgame.git'
    REPO_BRANCH      = 'new'

    SONARQUBE_SERVER = 'SonarQube'                 // Manage Jenkins → Configure System

    // Nexus 2 base URL (note the /nexus)
    NEXUS_URL        = 'http://18.188.63.155:8081/nexus'
    NEXUS_CRED_ID    = 'nexus-cred'                // Jenkins Credentials (Username/Password)

    // Tomcat deploy target
    TOMCAT_HOST      = '18.220.246.223'
    TOMCAT_USER      = 'ec2-user'
    TOMCAT_SSH_ID    = 'tomcat-ssh'                // Jenkins SSH key ID
    TOMCAT_WEBAPPS   = '/opt/tomcat/webapps'
    APP_NAME         = 'NumberGuessGame'
  }

  stages {
    stage('Checkout') {
      steps {
        timestamps {
          git branch: env.REPO_BRANCH, url: env.REPO_URL
        }
      }
    }

    stage('Build') {
      steps {
        timestamps {
          sh 'mvn -B -DskipTests clean package'
        }
      }
    }

    stage('SonarQube Scan') {
      steps {
        timestamps {
          withSonarQubeEnv(env.SONARQUBE_SERVER) {
            sh 'mvn -B sonar:sonar'
          }
        }
      }
    }

    stage('Publish to Nexus (Nexus 2)') {
      steps {
        timestamps {
          script {
            // Decide snapshots vs releases from pom version
            def version = sh(script: "mvn -q -DforceStdout help:evaluate -Dexpression=project.version", returnStdout: true).trim()
            def targetRepo = version.endsWith('-SNAPSHOT') ? 'snapshots' : 'releases'
            echo "Project version: ${version} → deploying to '${targetRepo}'"

            withCredentials([usernamePassword(credentialsId: env.NEXUS_CRED_ID, usernameVariable: 'NU', passwordVariable: 'NP')]) {
              // Minimal settings.xml so Maven authenticates cleanly
              writeFile file: 'settings.xml', text: """
<settings>
  <servers>
    <server>
      <id>${targetRepo}</id>
      <username>${NU}</username>
      <password>${NP}</password>
    </server>
  </servers>
</settings>
"""

              // Quick preflight (prints only HTTP status line)
              sh """
                set -e
                curl -sS -I -u "$NU:$NP" "${NEXUS_URL}/content/repositories/${targetRepo}/" | head -n1
                echo "Deploy URL: ${NEXUS_URL}/content/repositories/${targetRepo}/"
                mvn -B -s settings.xml -DskipTests deploy \\
                  -DaltDeploymentRepository=${targetRepo}::default::${NEXUS_URL}/content/repositories/${targetRepo}/
                rm -f settings.xml
              """
            }
          }
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        timestamps {
          script {
            def war = sh(script: "ls -1 target/*.war | head -n1", returnStdout: true).trim()
            sshagent([env.TOMCAT_SSH_ID]) {
              sh """
                set -e
                scp -o StrictHostKeyChecking=no "${war}" ${TOMCAT_USER}@${TOMCAT_HOST}:/tmp/app.war
                ssh -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_HOST} '
                  set -e
                  sudo rm -f ${TOMCAT_WEBAPPS}/${APP_NAME}.war || true
                  sudo rm -rf ${TOMCAT_WEBAPPS}/${APP_NAME} || true
                  sudo cp /tmp/app.war ${TOMCAT_WEBAPPS}/${APP_NAME}.war
                '
              """
            }
          }
        }
      }
    }
  }

  post {
    success { echo 'Build, scan, publish, and deploy completed.' }
    failure { echo 'Pipeline failed. Check logs for the failing stage.' }
  }
}
