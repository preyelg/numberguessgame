pipeline {
  agent any

  // MUST match names in Manage Jenkins → Tools
  tools {
    jdk   'java-17'
    maven 'Maven'
  }

  environment {
    REPO_URL          = 'https://github.com/preyelg/numberguessgame.git'
    REPO_BRANCH       = 'new'

    // SonarQube (name must match Manage Jenkins → Configure System)
    SONARQUBE_SERVER  = 'SonarQube'

    // Remote Tomcat host
    TOMCAT_HOST       = '18.220.246.223'
    TOMCAT_USER       = 'ec2-user'
    TOMCAT_SSH_ID     = 'tomcat-ssh'    // Jenkins credential: SSH Username with private key
    APP_NAME          = 'NumberGuessGame'
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
          // Optional Quality Gate enforcement (requires Sonar webhook to Jenkins)
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
            // Pick the most recent WAR produced by the build
            def war = sh(script: "ls -1 target/*.war | tail -n1", returnStdout: true).trim()
            if (!war) { error 'No WAR found under target/. Did the build stage run?' }
            echo "Deploying WAR: ${war}"

            // Requires SSH Agent plugin + 'tomcat-ssh' credential
            sshagent(credentials: [env.TOMCAT_SSH_ID]) {
              sh """
                set -e

                # Upload to /tmp on the remote host
                scp -o StrictHostKeyChecking=no "${war}" ${env.TOMCAT_USER}@${env.TOMCAT_HOST}:/tmp/${env.APP_NAME}.war

                # Run deployment logic remotely; escape all \$ so Groovy won't interpolate them
                ssh -o StrictHostKeyChecking=no ${env.TOMCAT_USER}@${env.TOMCAT_HOST} 'bash -s' <<'EOS'
set -e

# 1) Find Tomcat webapps directory (try common locations/globs)
CANDIDATES=(
  /opt/tomcat/webapps
  /usr/share/tomcat/webapps
  /usr/share/tomcat*/webapps
  /usr/local/tomcat/webapps
  /var/lib/tomcat/webapps
  /var/lib/tomcat*/webapps
)

WEBAPPS_DIR=""
for d in "\${CANDIDATES[@]}"; do
  for p in \$d; do
    if [ -d "\$p" ]; then WEBAPPS_DIR="\$p"; break 2; fi
  done
done

if [ -z "\$WEBAPPS_DIR" ]; then
  echo "ERROR: Could not locate Tomcat webapps directory on \$(hostname)." >&2
  echo "Hint: run 'sudo find / -maxdepth 3 -type d -name webapps 2>/dev/null' to locate it." >&2
  exit 1
fi
echo "Using WEBAPPS_DIR: \$WEBAPPS_DIR"

# 2) Drop new WAR
sudo rm -f  "\$WEBAPPS_DIR/${APP_NAME}.war" || true
sudo rm -rf "\$WEBAPPS_DIR/${APP_NAME}"     || true
sudo cp /tmp/${APP_NAME}.war "\$WEBAPPS_DIR/${APP_NAME}.war"

# 3) Fix ownership if a 'tomcat' user exists (best-effort)
if id tomcat >/dev/null 2>&1; then
  sudo chown tomcat:tomcat "\$WEBAPPS_DIR/${APP_NAME}.war" || true
fi

# 4) Restart Tomcat if a known service exists; else rely on auto-deploy
set +e
for svc in tomcat tomcat9 tomcat8 tomcat7; do
  if systemctl list-unit-files | grep -q "^\$svc\\.service"; then
    echo "Restarting service: \$svc (systemd)"
    sudo systemctl restart "\$svc" && exit 0
  fi
  if command -v service >/dev/null 2>&1 && service "\$svc" status >/dev/null 2>&1; then
    echo "Restarting service: \$svc (SysV)"
    sudo service "\$svc" restart && exit 0
  fi
done
echo "No known Tomcat service found; assuming auto-deploy is enabled."
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
    failure { echo 'Pipeline failed. Check logs above for the failing stage.' }
  }
}
