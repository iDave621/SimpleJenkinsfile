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
  - name: git
    image: alpine/git:2.45.2
    command: ['sh','-c','cat']
    tty: true
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command: ['sh','-c','cat']
    tty: true
"""
    }
  }

  options {
    disableConcurrentBuilds()
    timestamps()
  }

  parameters {
    string(name: 'REPO_URL',      defaultValue: '', description: 'Git repository URL')
    string(name: 'GIT_REF',       defaultValue: 'main', description: 'Branch / tag / commit')
    string(name: 'PROJECT_NAME',  defaultValue: '', description: 'Microservice folder name')
    string(name: 'ECR_REGISTRY',  defaultValue: '', description: 'ECR registry')
    string(name: 'ECR_REPO',      defaultValue: '', description: 'ECR repository name')
    string(name: 'IMAGE_TAG',     defaultValue: '', description: 'Optional custom image tag')
  }

  environment {
    SRC_DIR        = "src"
    DOCKERFILE     = "Dockerfile"
    BUILD_CONTEXT  = ""
    FULL_IMAGE     = ""
    FINAL_TAG      = ""
    GIT_COMMIT_SHA = ""
  }

  stages {

    stage('Init Environment') {
      steps {
        script {
          env.SERVICE_NAME = params.PROJECT_NAME
          env.BUILD_CONTEXT = "${env.SRC_DIR}/${env.SERVICE_NAME}"
          env.FINAL_TAG = params.IMAGE_TAG?.trim()
            ? params.IMAGE_TAG
            : "${env.SERVICE_NAME}-${env.BUILD_NUMBER}"

          env.FULL_IMAGE = "${params.ECR_REGISTRY}/${params.ECR_REPO}:${env.FINAL_TAG}"
        }
      }
    }

    stage('Checkout Source') {
      steps {
        container('git') {
          sh """
            set -eux
            rm -rf ${SRC_DIR}
            git clone ${params.REPO_URL} ${SRC_DIR}
            cd ${SRC_DIR}
            git checkout ${params.GIT_REF}
            git rev-parse HEAD > /tmp/commit
          """

          script {
            env.GIT_COMMIT_SHA = sh(
              script: "cat /tmp/commit",
              returnStdout: true
            ).trim()
          }
        }
      }
    }

    stage('Validate Inputs') {
      steps {
        container('git') {
          sh """
            set -e

            test -n "${SERVICE_NAME}"
            test -d ${BUILD_CONTEXT}
            test -f ${BUILD_CONTEXT}/${DOCKERFILE}
          """

          echo """
CI Build Context
----------------
Service Name   : ${SERVICE_NAME}
Git Ref        : ${params.GIT_REF}
Commit SHA     : ${GIT_COMMIT_SHA}
Build Context  : ${BUILD_CONTEXT}
Dockerfile     : ${DOCKERFILE}
Image          : ${FULL_IMAGE}
"""
        }
      }
    }

    stage('Build Image') {
      steps {
        container('kaniko') {
          sh """
            set -eux

            /kaniko/executor \
              --context dir://${WORKSPACE}/${BUILD_CONTEXT} \
              --dockerfile ${WORKSPACE}/${BUILD_CONTEXT}/${DOCKERFILE} \
              --destination ${FULL_IMAGE} \
              --no-push \
              --cache=true
          """
        }
      }
    }

    stage('Push Image') {
      steps {
        container('kaniko') {
          sh """
            set -eux

            /kaniko/executor \
              --context dir://${WORKSPACE}/${BUILD_CONTEXT} \
              --dockerfile ${WORKSPACE}/${BUILD_CONTEXT}/${DOCKERFILE} \
              --destination ${FULL_IMAGE} \
              --cache=true
          """
        }
      }
    }

    stage('Summary') {
      steps {
        echo """
✅ CI completed successfully

Service     : ${SERVICE_NAME}
Repository  : ${params.REPO_URL}
Commit      : ${GIT_COMMIT_SHA}
Image       : ${FULL_IMAGE}
"""
      }
    }
  }
}