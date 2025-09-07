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

    // SonarQube server name as configured in Manage Jenkins → Configure System
    SONARQUBE_SERVER = 'SonarQube'

    // ---- Tomcat settings ----
    TOMCAT_HOST      = '18.220.246.223'
    TOMCAT_USER      = 'ec2-user'
    TOMCAT_SSH_ID    = 'tomcat-ssh'    // SSH Username with private key (Jenkins credential)
    TOMCAT_WEBAPPS   = '/home/ec2-user/apache-tomcat-7.0.94/webapps'
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
          // Optional: enforce Quality Gate (requires Sonar webhook to Jenkins)
          // timeout(time: 10, unit: 'MINUTES') { waitForQualityGate abortPipeline: true }
        }
      }
    }

    stage('Deploy to Tomcat') {
      steps {
        timestamps {
          script {
            def war = sh(script: "ls -1 target/*.war | tail -n1", returnStdout: true).trim()
            if (!war) { error 'No WAR found under target/ — did the build run?' }
            echo "Deploying WAR: ${war}"

            sshagent(credentials: [env.TOMCAT_SSH_ID]) {
              sh """
                set -e
                # Upload to remote /tmp
                scp -o StrictHostKeyChecking=no "${war}" ${env.TOMCAT_USER}@${env.TOMCAT_HOST}:/tmp/${env.APP_NAME}.war

                # Remote deploy
                ssh -o StrictHostKeyChecking=no ${env.TOMCAT_USER}@${env.TOMCAT_HOST} 'bash -s' <<EOS
set -e

WEBAPPS_DIR="${env.TOMCAT_WEBAPPS}"
APP="${env.APP_NAME}"

echo "Using WEBAPPS_DIR: \$WEBAPPS_DIR"

# Drop new WAR
rm -f  "\$WEBAPPS_DIR/\$APP.war" || true
rm -rf "\$WEBAPPS_DIR/\$APP"     || true
mv /tmp/\$APP.war "\$WEBAPPS_DIR/\$APP.war"

# Try system service first
set +e
for svc in tomcat tomcat9 tomcat8 tomcat7; do
  if command -v systemctl >/dev/null 2>&1 && systemctl list-unit-files | grep -q "^\$svc\\.service"; then
    echo "Restarting via systemd: \$svc"
    sudo systemctl restart "\$svc" && exit 0
  fi
  if command -v service >/dev/null 2>&1 && service "\$svc" status >/dev/null 2>&1; then
    echo "Restarting via service: \$svc"
    sudo service "\$svc" restart && exit 0
  fi
done

# Fall back to startup/shutdown scripts next to webapps
CATALINA_BIN="\$(dirname "\$WEBAPPS_DIR")/bin"
if [ -x "\$CATALINA_BIN/shutdown.sh" ] && [ -x "\$CATALINA_BIN/startup.sh" ]; then
  echo "Restarting via scripts in \$CATALINA_BIN"
  "\$CATALINA_BIN/shutdown.sh" || true
  sleep 5
  "\$CATALINA_BIN/startup.sh" || true
else
  echo "No service or scripts found — relying on Tomcat auto-deploy."
fi
exit 0
EOS
              """
            }
          }
        }
      }
    }
  }

  post {
    success { echo 'Build, Sonar scan, and Tomcat deploy completed.' }
    failure { echo 'Pipeline failed — check the stage logs above.' }
  }
}
