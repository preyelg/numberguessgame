pipeline {
  agent any

  // MUST match names in Manage Jenkins → Tools
  tools {
    jdk   'java-17'
    maven 'Maven'
  }

  environment {
    REPO_URL       = 'https://github.com/preyelg/numberguessgame.git'
    REPO_BRANCH    = 'new'

    // SonarQube (name must match Manage Jenkins → Configure System)
    SONARQUBE_SERVER = 'SonarQube'

    // Tomcat target
    TOMCAT_HOST    = '18.220.246.223'
    TOMCAT_USER    = 'ec2-user'
    TOMCAT_SSH_ID  = 'tomcat-ssh'          // Jenkins credential: SSH Username with private key
    TOMCAT_WEBAPPS = '/opt/tomcat/webapps'
    TOMCAT_SERVICE = 'tomcat'              // change if different (e.g., tomcat9)
    APP_NAME       = 'NumberGuessGame'
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
          // Optional: enforce the Quality Gate (requires Sonar webhook to Jenkins)
          // timeout(time: 10, unit: 'MINUTES') {
          //   waitForQualityGate abortPipeline: true
          // }
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        timestamps {
          script {
            // pick the built WAR
            def war = sh(script: "ls -1 target/*.war | tail -n1", returnStdout: true).trim()
            if (!war) { error 'No WAR found under target/. Did the build stage run?' }
            echo "Deploying WAR: ${war}"

            // Requires SSH Agent plugin + 'tomcat-ssh' credential
            sshagent (credentials: [env.TOMCAT_SSH_ID]) {
              sh """
                set -e
                scp -o StrictHostKeyChecking=no "${war}" ${TOMCAT_USER}@${TOMCAT_HOST}:/tmp/${APP_NAME}.war
                ssh -o StrictHostKeyChecking=no ${TOMCAT_USER}@${TOMCAT_HOST} '
                  set -e
                  sudo rm -f ${TOMCAT_WEBAPPS}/${APP_NAME}.war || true
                  sudo rm -rf ${TOMCAT_WEBAPPS}/${APP_NAME} || true
                  sudo cp /tmp/${APP_NAME}.war ${TOMCAT_WEBAPPS}/${APP_NAME}.war
                  sudo chown -f tomcat:tomcat ${TOMCAT_WEBAPPS}/${APP_NAME}.war || true
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
    success { echo 'Build, Sonar scan, and Tomcat deploy completed.' }
    failure { echo 'Pipeline failed. Check logs above for the failing stage.' }
  }
}
