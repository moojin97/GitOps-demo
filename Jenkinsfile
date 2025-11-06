pipeline {
  agent {
    kubernetes {
      defaultContainer 'jnlp'
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: default

  # harbor.local 을 192.168.25.100 으로 고정 해석 (파드 /etc/hosts 주입)
  hostAliases:
  - ip: "192.168.25.100"
    hostnames:
    - "harbor.local"

  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug   # BusyBox 포함 이미지
    command: ["/busybox/sh","-c","sleep 999999"]  # 대기 (stage에서 sh 실행)
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
    // 레지스트리/레포 (TLS SNI는 harbor.local, 실주소는 hostAliases로 매핑)
    IMAGE_REPO     = "harbor.local/demo/nginx-sample"
    GIT_URL        = "https://github.com/moojin97/GitOps-demo.git"
    GIT_USER_NAME  = "moojin97"
    GIT_USER_EMAIL = "you@example.com"
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
          # tag 줄만 안전 치환
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
