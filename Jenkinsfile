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

  parameters {
    string(
      name: 'REPO_URL',
      defaultValue: 'https://github.com/iDave621/AWS-Luxe-Jewelry-Store',
      description: 'Git repository to build from'
    )
    string(
      name: 'ECR_REGISTRY',
      defaultValue: '553817834164.dkr.ecr.us-east-1.amazonaws.com',
      description: 'ECR registry'
    )
    string(
      name: 'ECR_REPO',
      defaultValue: 'luxe-jewelry-store',
      description: 'Single ECR repository name'
    )
  }

  options {
    disableConcurrentBuilds()
  }

  environment {
    SRC_DIR = "src"
  }

  stages {

    stage('Checkout Repo') {
      steps {
        container('git') {
          sh """
            set -eux
            rm -rf ${SRC_DIR}
            git clone ${params.REPO_URL} ${SRC_DIR}
            cd ${SRC_DIR}
            git fetch --tags --force
          """
        }
      }
    }

    stage('Detect Tag') {
      steps {
        container('git') {
          script {
            String ref = (env.GIT_REF ?: "").trim()

            if (!ref) {
              ref = sh(
                script: """
                  set -e
                  cd ${SRC_DIR}
                  git for-each-ref --sort=-creatordate \
                    --format '%(refname:short)' refs/tags | head -n 1
                """,
                returnStdout: true
              ).trim()

              if (!ref) {
                error "No tag found and no webhook ref provided"
              }

              ref = "refs/tags/${ref}"
            }

            if (!ref.startsWith("refs/tags/")) {
              error "Not a tag ref: ${ref}"
            }

            String tag = ref.replaceFirst("^refs/tags/", "").trim()

            // <service>-vX.Y.Z
            def m = (tag =~ /^(.+)-v(\d+\.\d+\.\d+)$/)
            if (!m.matches()) {
              error "Invalid tag format. Expected <service>-vX.Y.Z , got ${tag}"
            }

            env.SERVICE_NAME = m[0][1]
            env.BASE_VERSION = m[0][2]
            env.FINAL_VERSION = "${env.BASE_VERSION}.${env.BUILD_NUMBER}"
            env.TAG_NAME = tag

            echo """
Triggered tag : ${env.TAG_NAME}
Service       : ${env.SERVICE_NAME}
Base version  : ${env.BASE_VERSION}
Build number  : ${env.BUILD_NUMBER}
Final version : ${env.FINAL_VERSION}
""".stripIndent()
          }
        }
      }
    }

    stage('Resolve Service Path') {
      steps {
        container('git') {
          script {
            String svcPath = env.SERVICE_NAME

            def exists = sh(
              script: "set -e; test -d ${SRC_DIR}/${svcPath} && echo OK || echo NO",
              returnStdout: true
            ).trim()

            if (exists != "OK") {
              error "Service folder not found: ${SRC_DIR}/${svcPath}"
            }

            env.SVC_PATH = svcPath
            env.SVC_DOCKERFILE = "Dockerfile"

            echo "Building service '${env.SERVICE_NAME}' from '${env.SVC_PATH}'"
          }
        }
      }
    }

    stage('Build & Push Image') {
      steps {
        container('kaniko') {
          script {
            env.IMAGE = "${params.ECR_REGISTRY}/${params.ECR_REPO}"
            env.IMAGE_TAG = "${env.SERVICE_NAME}-${env.FINAL_VERSION}"
          }

          sh """
            set -eux

            echo "IMAGE       = ${IMAGE}"
            echo "TAG         = ${IMAGE_TAG}"
            echo "CONTEXT     = ${WORKSPACE}/${SRC_DIR}/${SVC_PATH}"
            echo "DOCKERFILE  = ${WORKSPACE}/${SRC_DIR}/${SVC_PATH}/${SVC_DOCKERFILE}"

            /kaniko/executor \
              --context dir://${WORKSPACE}/${SRC_DIR}/${SVC_PATH} \
              --dockerfile ${WORKSPACE}/${SRC_DIR}/${SVC_PATH}/${SVC_DOCKERFILE} \
              --destination ${IMAGE}:${IMAGE_TAG} \
              --cache=true
          """
        }
      }
    }

    stage('Summary') {
      steps {
        echo "✅ Pushed image:"
        echo "   ${env.IMAGE}:${env.IMAGE_TAG}"
      }
    }
  }
}
