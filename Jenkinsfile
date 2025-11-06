pipeline {
  agent {
    kubernetes {
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: default
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:latest
    args: ["sleep", "999999"]
    volumeMounts:
    - name: docker-config
      mountPath: /kaniko/.docker
  volumes:
  - name: docker-config
    projected:
      sources:
      - secret:
          name: regcred
          items:
          - key: .dockerconfigjson
            path: config.json
"""
    }
  }

  environment {
    // ❗ 본인 Harbor 경로에 맞게 맞춰둠 (443, LB 사용)
    IMAGE_REPO    = "harbor.local/demo/nginx-sample"
    GIT_URL       = "https://github.com/moojin97/GitOps-demo.git"
    GIT_USER_NAME = "moojin97"
    GIT_USER_EMAIL= "you@example.com"
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Push (Kaniko)') {
      steps {
        container('kaniko') {
          sh '''
            TAG=build-${BUILD_NUMBER}
            /kaniko/executor \
              --context `pwd`/app \
              --dockerfile `pwd`/app/Dockerfile \
              --destination ${IMAGE_REPO}:$TAG
            echo $TAG > .last_tag
          '''
        }
      }
    }

    stage('Update Helm values.yaml & Commit') {
      steps {
        sh '''
          TAG=$(cat .last_tag)
          V=helm/nginx-sample/values.yaml
          sed -i "s/^  tag: .*/  tag: \\"$TAG\\"/" $V
          git config user.name  "${GIT_USER_NAME}"
          git config user.email "${GIT_USER_EMAIL}"
          git add $V
          git commit -m "ci: update image tag to $TAG" || true
        '''
      }
    }

    stage('Push to Git (Trigger ArgoCD)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'git-cred', usernameVariable: 'GIT_U', passwordVariable: 'GIT_P')]) {
          sh '''
            git push https://${GIT_U}:${GIT_P}@${GIT_URL#https://} HEAD:main
          '''
        }
      }
    }
  }
}
