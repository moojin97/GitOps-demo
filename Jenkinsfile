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
    # ⚠️ executor 이미지엔 /bin/sleep 없음 → /busybox/sh 로 대기
    command:
      - /busybox/sh
      - -c
      - sleep 999999
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
    // ← 네 환경에 맞춰 이미 설정됨
    IMAGE_REPO     = "harbor.local/demo/nginx-sample"
    GIT_URL        = "https://github.com/moojin97/GitOps-demo.git"
    GIT_USER_NAME  = "moojin97"
    GIT_USER_EMAIL = "you@example.com"   // 원하면 사용 메일로 교체
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Push (Kaniko)') {
      steps {
        container('kaniko') {
          sh '''
            set -e
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
          set -e
          TAG=$(cat .last_tag)
          V=helm/nginx-sample/values.yaml
          # tag 줄만 안전하게 치환
          sed -i "s/^\\s*tag:\\s*.*/  tag: \\"$TAG\\"/" $V
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
            set -e
            git push https://${GIT_U}:${GIT_P}@${GIT_URL#https://} HEAD:main
          '''
        }
      }
    }
  }
}
