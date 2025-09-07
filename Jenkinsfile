pipeline {
  agent any

  // Make sure these names MATCH Manage Jenkins → Tools
  tools { 
    jdk   'java-17'
    maven 'Maven'
  }

  environment {
    REPO_URL        = 'https://github.com/preyelg/numberguessgame.git'
    REPO_BRANCH     = 'new'
    SONARQUBE_SERVER= 'SonarQube'         // Must match Manage Jenkins → Configure System name
    NEXUS_URL       = 'http://18.188.63.155:8081/nexus'
    NEXUS_REPO      = 'https://github.com/preyelg/numberguessgame.git'
    NEXUS_CRED_ID   = 'nexus-cred'

    TOMCAT_HOST     = '18.220.246.223'
    TOMCAT_USER     = 'ec2-user'
    TOMCAT_SSH_ID   = 'tomcat-ssh'
    TOMCAT_WEBAPPS  = '/opt/tomcat/webapps'
    APP_NAME        = 'NumberGuessGame'
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

    stage('Publish to Nexus') {
      steps {
        timestamps {
          withCredentials([usernamePassword(credentialsId: env.NEXUS_CRED_ID, usernameVariable: 'NU', passwordVariable: 'NP')]) {
            sh """
              mvn -B -DskipTests deploy \
                -DaltDeploymentRepository=${NEXUS_REPO}::default::${NEXUS_URL}/repository/${NEXUS_REPO}/ \
                -DrepositoryId=${NEXUS_REPO} \
                -Durl=${NEXUS_URL}/repository/${NEXUS_REPO}/ \
                -Dusername=$NU -Dpassword=$NP
            """
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
