pipeline {
  agent any

  environment {
    DOCKERHUB_REPO = 'your-dockerhub-username/your-repo'   // e.g. "alice/flask-ec2"
    IMAGE_TAG      = "${env.BUILD_NUMBER}"
    EC2_SSH_USER   = 'ec2-user'                             // or ubuntu for Ubuntu AMIs
    EC2_HOST       = 'YOUR.EC2.PUBLIC.IP'                   // or DNS
    APP_NAME       = 'flask-ec2-app'
    CONTAINER_PORT = '5000'
    HOST_PORT      = '80'                                   // serve on :80
  }

  options {
    timestamps()
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Unit tests') {
      steps {
        sh '''
          python3 -m venv venv
          . venv/bin/activate
          pip install -r requirements.txt
          python - <<'PY'
print("No tests yet — add pytest later.")
PY
        '''
      }
    }

    stage('Docker Build') {
      steps {
        sh "docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} ."
        sh "docker tag ${DOCKERHUB_REPO}:${IMAGE_TAG} ${DOCKERHUB_REPO}:latest"
      }
    }

    stage('Docker Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push '${DOCKERHUB_REPO}:${IMAGE_TAG}'
            docker push '${DOCKERHUB_REPO}:latest'
          '''
        }
      }
    }

    stage('Deploy to EC2') {
      steps {
        sshagent(credentials: ['ec2-ssh-key']) {
          sh """
            ssh -o StrictHostKeyChecking=no ${EC2_SSH_USER}@${EC2_HOST} 'sudo docker login -u ${DOCKERHUB_REPO%/*} -p $(echo ${DOCKERHUB_REPO%/*}) >/dev/null 2>&1 || true'
            ssh -o StrictHostKeyChecking=no ${EC2_SSH_USER}@${EC2_HOST} '
              set -e
              sudo docker pull ${DOCKERHUB_REPO}:${IMAGE_TAG}
              # stop and remove any old container
              if sudo docker ps -a --format "{{.Names}}" | grep -q "^${APP_NAME}\$"; then
                sudo docker rm -f ${APP_NAME} || true
              fi
              # run new container
              sudo docker run -d --name ${APP_NAME} --restart unless-stopped \
                -p ${HOST_PORT}:${CONTAINER_PORT} \
                ${DOCKERHUB_REPO}:${IMAGE_TAG}
              # prune old images (optional)
              sudo docker image prune -f
            '
          """
        }
      }
    }
  }

  post {
    success {
      echo "Deployed ${DOCKERHUB_REPO}:${IMAGE_TAG} to ${EC2_HOST}:${HOST_PORT}"
    }
    failure {
      echo "Pipeline failed. Check logs."
    }
  }
}
