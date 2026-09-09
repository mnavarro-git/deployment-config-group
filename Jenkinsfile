pipeline {
  agent {
    kubernetes {
      yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - sleep
    args:
    - 99d
    resources:
      requests:
        memory: "1Gi"
      limits:
        memory: "2Gi"
  - name: git
    image: alpine/git:latest
    command:
    - sleep
    args:
    - 99d
'''
    }
  }

  environment {
    DOCKERHUB_CREDS = credentials('dockerhub-creds')
    GITHUB_CREDS = credentials('github-creds')
    IMAGE_NAME = "python-flaskapp"
    DEPLOY_REPO = "github.com/mnavarro-git/deployment-config.git"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script {
          env.GIT_COMMIT_SHORT = sh(
            script: "git rev-parse --short HEAD",
            returnStdout: true
          ).trim()
        }
      }
    }

    stage('Build and Push') {
      steps {
        container('kaniko') {
          sh '''
            mkdir -p /kaniko/.docker
            AUTH=$(echo -n "$DOCKERHUB_CREDS_USR:$DOCKERHUB_CREDS_PSW" | base64)
            cat > /kaniko/.docker/config.json << CONFIGEOF
{
  "auths": {
    "https://index.docker.io/v1/": {
      "auth": "$AUTH"
    }
  }
}
CONFIGEOF
            /kaniko/executor \
              --context=`pwd` \
              --dockerfile=Dockerfile \
              --build-arg APP_VERSION=$GIT_COMMIT_SHORT \
              --destination=$DOCKERHUB_CREDS_USR/$IMAGE_NAME:$GIT_COMMIT_SHORT
          '''
        }
      }
    }

    stage('Update deployment-config') {
      steps {
        container('git') {
          sh '''
            rm -rf deploy-config-clone
            git clone https://$GITHUB_CREDS_USR:$GITHUB_CREDS_PSW@$DEPLOY_REPO deploy-config-clone
            cd deploy-config-clone

            NEW_IMAGE="$DOCKERHUB_CREDS_USR/$IMAGE_NAME:$GIT_COMMIT_SHORT"

            grep -q "image:" base/deployment.yaml || { echo "ERROR: image field not found"; exit 1; }
            sed -i "s|image:.*|image: $NEW_IMAGE|" base/deployment.yaml

            git config user.email "jenkins@academy.local"
            git config user.name "jenkins-bot"

            git add base/deployment.yaml

            if git diff --cached --quiet; then
              echo "No change in image tag — nothing to commit."
            else
              git commit -m "Update image to $NEW_IMAGE"
              git push origin main
            fi
          '''
        }
      }
    }
  }
}
