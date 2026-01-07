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
  }

  parameters {
    string(name: 'REPO_URL',     description: 'Git repository URL')
    string(name: 'GIT_REF',      description: 'Git tag: <project>-vX.Y.Z')
    string(name: 'PROJECT_NAME', description: 'Microservice name (folder + ECR repo)')
    string(name: 'ECR_REGISTRY', description: 'ECR registry (account.dkr.ecr.region.amazonaws.com)')
  }

  environment {
    SRC_DIR       = "src"
    SERVICE_NAME  = ""
    GIT_TAG       = ""
    VERSION       = ""
    FINAL_TAG     = ""
    BUILD_CONTEXT = ""
    IMAGE_NAME    = ""
    FULL_IMAGE    = ""
  }

  stages {

    stage('Init & Resolve Metadata') {
      steps {
        container('git') {
          script {
            env.SERVICE_NAME = params.PROJECT_NAME?.trim()
            env.GIT_TAG      = params.GIT_REF?.trim()

            if (!env.SERVICE_NAME) {
              error "PROJECT_NAME is required"
            }

            if (!env.GIT_TAG) {
              error "GIT_REF is required"
            }

            String expectedPrefix = "${env.SERVICE_NAME}-"

            if (!env.GIT_TAG.startsWith(expectedPrefix)) {
              error "Git tag '${env.GIT_TAG}' must start with '${expectedPrefix}'"
            }

            env.VERSION = env.GIT_TAG.substring(expectedPrefix.length())

            if (!env.VERSION.startsWith("v")) {
              error "Version '${env.VERSION}' must start with 'v'"
            }

            env.FINAL_TAG     = "${env.VERSION}.${env.BUILD_NUMBER}"
            env.BUILD_CONTEXT = "${env.SRC_DIR}/${env.SERVICE_NAME}"
            env.IMAGE_NAME    = env.SERVICE_NAME
            env.FULL_IMAGE    = "${params.ECR_REGISTRY}/${env.IMAGE_NAME}:${env.FINAL_TAG}"

            echo """
Resolved Build Info
-------------------
Project        : ${env.SERVICE_NAME}
Git Tag        : ${env.GIT_TAG}
Base Version   : ${env.VERSION}
Build Number   : ${env.BUILD_NUMBER}
Final Tag      : ${env.FINAL_TAG}
ECR Repository : ${env.IMAGE_NAME}
Image          : ${env.FULL_IMAGE}
"""
          }
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
            git checkout ${env.GIT_TAG}
          """
        }
      }
    }

    stage('Validate Build Context') {
      steps {
        container('git') {
          sh """
            set -e
            test -d ${BUILD_CONTEXT}
            test -f ${BUILD_CONTEXT}/Dockerfile
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
              --dockerfile ${WORKSPACE}/${BUILD_CONTEXT}/Dockerfile \
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
              --dockerfile ${WORKSPACE}/${BUILD_CONTEXT}/Dockerfile \
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

Service : ${SERVICE_NAME}
Image   : ${FULL_IMAGE}
"""
      }
    }
  }
}