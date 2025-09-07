pipeline {
  agent any

  // MUST match Manage Jenkins → Tools
  tools {
    jdk   'java-17'
    maven 'Maven'
  }

  environment {
    REPO_URL         = 'https://github.com/preyelg/numberguessgame.git'
    REPO_BRANCH      = 'new1'

    SONARQUBE_SERVER = 'SonarQube'      // Manage Jenkins → Configure System

    // Nexus 2 base URL (note the /nexus)
    NEXUS_URL        = 'http://18.188.63.155:8081/nexus'
    NEXUS_CRED_ID    = 'nexus-cred'     // Jenkins credentials ID (Username/Password)

    // Will be set dynamically to 'snapshots' or 'releases'
    NEXUS_TARGET_REPO = ''
    
    // Tomcat deploy target
    TOMCAT_HOST      = '18.220.246.223'
    TOMCAT_USER      = 'ec2-user'
    TOMCAT_SSH_ID    = 'tomcat-ssh'     // Jenkins SSH credentials ID
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

    stage('Determine Target Repo') {
      steps {
        timestamps {
          script {
            // Read version from pom and choose snapshots vs releases
            def version = sh(script: "mvn -q -DforceStdout help:evaluate -Dexpression=project.version", returnStdout: true).trim()
            env.NEXUS_TARGET_REPO = version.endsWith('-SNAPSHOT') ? 'snapshots' : 'releases'
            echo "Project version: ${version} → deploying to '${env.NEXUS_TARGET_REPO}'"
          }
        }
      }
    }

    stage('Publish to Nexus') {
      steps {
        timestamps {
          withCredentials([usernamePassword(credentialsId: env.NEXUS_CRED_ID, usernameVariable: 'NU', passwordVariable: 'NP')]) {
            // Single quotes avoid Groovy interpolation of secrets; variables expand in bash
            sh '''
              set -e
              echo "Deploy URL: $NEXUS_URL/content/repositories/$NEXUS_TARGET_REPO/"
              mvn -B -DskipTests deploy \
                -DaltDeploymentRepository=${NEXUS_TARGET_REPO}::default::${NEXUS_URL}/content/repositories/${NEXUS_TARGET_REPO}/ \
                -DrepositoryId=${NEXUS_TARGET_REPO} \
                -Durl=${NEXUS_URL}/content/repositories/${NEXUS_TARGET_REPO}/ \
                -Dusername="$NU" -Dpassword="$NP"
            '''
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
