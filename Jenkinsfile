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
              IMAGE_TAG="${IMAGE_TAG_PARAM:-$BUILD_NUMBER}"
              [ -z "$IMAGE_TAG" ] && IMAGE_TAG=latest

              mkdir -p /kaniko/.docker
              AUTH=$(printf "%s:%s" "$REG_USER" "$REG_PASS" | base64 | tr -d '\n')
              cat > /kaniko/.docker/config.json <<EOF
              {"auths":{"https://index.docker.io/v1/":{"auth":"$AUTH"}}}
EOF

              /kaniko/executor \
                --context "$PWD" \
                --dockerfile "$PWD/Dockerfile" \
                --destination "${IMAGE_REPOSITORY_PARAM}:${IMAGE_TAG}" \
                --destination "${IMAGE_REPOSITORY_PARAM}:latest"
            '''
          }
        }
      }
      environment {
        IMAGE_REPOSITORY_PARAM = "${params.IMAGE_REPOSITORY}"
        IMAGE_TAG_PARAM = "${params.IMAGE_TAG}"
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
            IMAGE_TAG="${IMAGE_TAG_PARAM:-$BUILD_NUMBER}"
            [ -z "$IMAGE_TAG" ] && IMAGE_TAG=latest

            helm upgrade --install "${RELEASE_NAME_PARAM}" "${HELM_CHART_PATH_PARAM}" \
              --namespace "${K8S_NAMESPACE_PARAM}" \
              --set image.repository="${IMAGE_REPOSITORY_PARAM}" \
              --set image.tag="${IMAGE_TAG}"
          '''
        }
      }
      environment {
        IMAGE_REPOSITORY_PARAM = "${params.IMAGE_REPOSITORY}"
        IMAGE_TAG_PARAM = "${params.IMAGE_TAG}"
        RELEASE_NAME_PARAM = "${params.RELEASE_NAME}"
        K8S_NAMESPACE_PARAM = "${params.K8S_NAMESPACE}"
        HELM_CHART_PATH_PARAM = "${params.HELM_CHART_PATH}"
      }
    }
  }
}
