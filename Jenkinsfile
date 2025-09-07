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

    // Comment this block out if you don't want Sonar at all
    SONARQUBE_SERVER = 'SonarQube'   // Manage Jenkins → Configure System

    // Tomcat deploy target
    TOMCAT_HOST      = '18.220.246.223'
    TOMCAT_USER      = 'ec2-user'
    TOMCAT_SSH_ID    = 'tomcat-ssh'  // Jenkins SSH credentials ID
    TOMCAT_WEBAPPS   = '/opt/tomcat/webapps'
    APP_NAME         = 'NumberGuessGame'
    TOMCAT_SERVICE   = 'tomcat'      // change if your service name differs
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

    // Remove this entire stage if you don’t want Sonar
    stage('SonarQube Scan') {
      steps {
        timestamps {
          withSonarQubeEnv(env.SONARQUBE_SERVER) {
            sh 'mvn -B sonar:sonar'
          }
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        timestamps {
          script {
            // pick the built WAR
            def war = sh(script: "ls -1 target/*.war | tail -n1", returnStdout: true).trim()
            if (!war) { error 'No WAR found under target/' }

            sshagent([env.TOMCAT_SSH_ID]) {
              sh """
                set -e
                echo "Uploading: ${war}"
                scp -o StrictHostKeyChecking=no "${war}" ${TOMCAT_USER}@${TOMCAT_HOST}:/tmp/app.war

                ssh -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_HOST} '
                  set -e
                  sudo rm -f ${TOMCAT_WEBAPPS}/${APP_NAME}.war || true
                  sudo rm -rf ${TOMCAT_WEBAPPS}/${APP_NAME} || true
                  sudo cp /tmp/app.war ${TOMCAT_WEBAPPS}/${APP_NAME}.war
                  sudo chown -f tomcat:tomcat ${TOMCAT_WEBAPPS}/${APP_NAME}.war || true
                  # try restart if your Tomcat needs it
                  (sudo systemctl restart ${TOMCAT_SERVICE} || sudo service ${TOMCAT_SERVICE} restart || true)
                '
              """
            }
          }
        }
      }
    }
  }

  post {
    success { echo 'Build and deploy to Tomcat completed.' }
    failure { echo 'Pipeline failed. Check the stage logs above.' }
  }
}
