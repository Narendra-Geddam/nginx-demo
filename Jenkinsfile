pipeline {
  agent none

  parameters {
    string(name: 'BRANCH', defaultValue: 'main', description: 'Git branch')
    string(name: 'IMAGE_REPOSITORY', defaultValue: 'privatergistry/nginx-demo', description: 'Docker image repo')
    string(name: 'IMAGE_TAG', defaultValue: '', description: 'Leave empty to use BUILD_NUMBER')
    string(name: 'RELEASE_NAME', defaultValue: 'nginx-demo', description: 'Helm release')
    string(name: 'K8S_NAMESPACE', defaultValue: 'dev', description: 'K8s namespace')
    string(name: 'HELM_CHART_PATH', defaultValue: 'helm/nginx-demo', description: 'Helm chart path')
  }

  environment {
    REGISTRY_CREDENTIALS_ID = 'dockerhub-creds'
  }

  stages {
    stage('Build & Push') {
      agent {
        kubernetes {
          yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:v1.23.2-debug
    command: ["/busybox/cat"]
    tty: true
"""
        }
      }
      steps {
        git branch: "${params.BRANCH}", url: 'https://github.com/Narendra-Geddam/nginx-demo.git'
        container('kaniko') {
          withCredentials([usernamePassword(credentialsId: env.REGISTRY_CREDENTIALS_ID, usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
            sh '''
              IMAGE_TAG="${PARAM_IMAGE_TAG:-$BUILD_NUMBER}"
              [ -z "$IMAGE_TAG" ] && IMAGE_TAG=latest

              mkdir -p /kaniko/.docker
              AUTH=$(printf "%s:%s" "$REG_USER" "$REG_PASS" | base64 | tr -d '\n')
              cat > /kaniko/.docker/config.json <<EOF
              {"auths":{"https://index.docker.io/v1/":{"auth":"$AUTH"}}}
EOF

              /kaniko/executor \
                --context "$PWD" \
                --dockerfile "$PWD/Dockerfile" \
                --destination "${PARAM_IMAGE_REPOSITORY}:${IMAGE_TAG}" \
                --destination "${PARAM_IMAGE_REPOSITORY}:latest"
            '''
          }
        }
      }
      environment {
        PARAM_IMAGE_REPOSITORY = "${params.IMAGE_REPOSITORY}"
        PARAM_IMAGE_TAG = "${params.IMAGE_TAG}"
      }
    }

    stage('Deploy') {
      agent {
        kubernetes {
          yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: helm
    image: alpine/helm:3.14.4
    command: ["/bin/sh","-c","cat"]
    tty: true
"""
        }
      }
      steps {
        git branch: "${params.BRANCH}", url: 'https://github.com/Narendra-Geddam/nginx-demo.git'
        container('helm') {
          sh '''
            IMAGE_TAG="${PARAM_IMAGE_TAG:-$BUILD_NUMBER}"
            [ -z "$IMAGE_TAG" ] && IMAGE_TAG=latest

            helm upgrade --install "${PARAM_RELEASE_NAME}" "${PARAM_HELM_CHART_PATH}" \
              --namespace "${PARAM_K8S_NAMESPACE}" \
              --set image.repository="${PARAM_IMAGE_REPOSITORY}" \
              --set image.tag="${IMAGE_TAG}"
          '''
        }
      }
      environment {
        PARAM_IMAGE_REPOSITORY = "${params.IMAGE_REPOSITORY}"
        PARAM_IMAGE_TAG = "${params.IMAGE_TAG}"
        PARAM_RELEASE_NAME = "${params.RELEASE_NAME}"
        PARAM_K8S_NAMESPACE = "${params.K8S_NAMESPACE}"
        PARAM_HELM_CHART_PATH = "${params.HELM_CHART_PATH}"
      }
    }
  }
}
