pipeline {

  agent {
    kubernetes {
      defaultContainer 'kaniko'
      yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins-sa
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest

  - name: git
    image: alpine/git:latest
    command: ['sh', '-c', 'cat']
    tty: true

  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: ['sh', '-c', 'cat']
    tty: true
"""
    }
  }

  environment {
    AWS_REGION      = 'us-east-1'
    AWS_ACCOUNT_ID = '553817834164'
    ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
  }

  stages {

    stage('Validate Tag Build') {
      steps {
        script {
          if (!env.GIT_TAG_NAME) {
            error("❌ CI runs only on Git tags")
          }

          // Strip leading 'v' if exists
          env.BASE_TAG = env.GIT_TAG_NAME.replaceFirst(/^v/, '')
          env.IMAGE_TAG = "${env.BASE_TAG}.${env.BUILD_NUMBER}"

          echo "🏷️ Git tag: ${env.GIT_TAG_NAME}"
          echo "🐳 Image tag: ${env.IMAGE_TAG}"
        }
      }
    }

    stage('Checkout Code') {
      steps {
        container('git') {
          checkout scm
        }
      }
    }

    stage('Detect Changed Services') {
      steps {
        script {
          def changedFiles = sh(
            script: "git diff --name-only HEAD~1 HEAD || true",
            returnStdout: true
          ).trim().split("\\n")

          def services = [:]

          if (changedFiles.any { it.startsWith("backend/") }) {
            services.backend = [
              path: 'backend',
              image: 'luxe-jewelry-store-backend'
            ]
          }

          if (changedFiles.any { it.startsWith("auth/") }) {
            services.auth = [
              path: 'auth',
              image: 'luxe-jewelry-store-auth'
            ]
          }

          if (changedFiles.any { it.startsWith("frontend/") }) {
            services.frontend = [
              path: 'frontend',
              image: 'luxe-jewelry-store-frontend'
            ]
          }

          if (services.isEmpty()) {
            echo "⚠️ No microservice changes detected"
            currentBuild.result = 'SUCCESS'
            return
          }

          env.SERVICES_TO_BUILD = groovy.json.JsonOutput.toJson(services)
        }
      }
    }

    stage('Build & Push Images (Kaniko)') {
      when {
        expression { env.SERVICES_TO_BUILD }
      }
      steps {
        script {
          def services = new groovy.json.JsonSlurperClassic()
            .parseText(env.SERVICES_TO_BUILD)

          def builds = [:]

          services.each { name, svc ->
            builds[name] = {
              container('kaniko') {
                sh """
                  echo "🚀 Building ${name}"

                  /kaniko/executor \
                    --context=${svc.path} \
                    --dockerfile=Dockerfile \
                    --destination=${ECR_REGISTRY}/${svc.image}:${IMAGE_TAG} \
                    --destination=${ECR_REGISTRY}/${svc.image}:latest \
                    --cache=true \
                    --cache-ttl=24h
                """
              }
            }
          }

          parallel builds
        }
      }
    }
  }

  post {
    success {
      echo "✅ CI finished successfully"
      echo "🏷️ Image tag: ${IMAGE_TAG}"
      echo "🚀 Argo CD will deploy automatically"
    }
    failure {
      echo "❌ CI failed"
    }
  }
}
