pipeline {
  agent any

  parameters {
    string(name: 'TAG_NAME', defaultValue: 'latest', description: 'Image tag from Pipeline 1')
  }

  environment {
    GIT_CREDENTIALS = 'github-token-or-userpass'  // Jenkins credential ID
    GIT_URL         = 'https://github.com/Kevinjosetom/kubernetesmanifest.git'
    GIT_BRANCH      = 'main'
    DEPLOYMENT_NAME = 'flaskdemo'
    CONTAINER_NAME  = 'myapp'
    IMAGE_REPO      = 'docker.io/kevinjosetom/myapp'
    TARGET_FILE     = 'deployment.yaml'           // change if your file is elsewhere
  }

  stages {
    stage('Checkout manifests repo') {
      steps {
        dir('manifests') {
          git branch: "${GIT_BRANCH}", url: "${GIT_URL}", credentialsId: "${GIT_CREDENTIALS}"
        }
      }
    }

    stage('Get yq (standalone binary)') {
      steps {
        dir('manifests') {
          sh '''
            set -eux
            ARCH=$(uname -m)
            case "$ARCH" in
              x86_64|amd64)  FILE=yq_linux_amd64 ;;
              aarch64|arm64) FILE=yq_linux_arm64 ;;
              armv7l)        FILE=yq_linux_arm ;;
              *)             FILE=yq_linux_amd64 ;;
            esac
            curl -L "https://github.com/mikefarah/yq/releases/latest/download/${FILE}" -o yq
            chmod +x yq
            ./yq --version
          '''
        }
      }
    }

    stage('Patch deployment YAML (file only)') {
      steps {
        dir('manifests') {
          sh """
            set -eux
            echo "Using TAG_NAME=${TAG_NAME}"
            ./yq -i '
              select(.kind=="Deployment" and .metadata.name=="${DEPLOYMENT_NAME}") |
              .spec.template.spec.containers[] |
              select(.name=="${CONTAINER_NAME}") |
              .image = "${IMAGE_REPO}:" + (env.TAG_NAME)
            ' "${TARGET_FILE}"
            git --no-pager diff -- "${TARGET_FILE}" || true
          """
        }
      }
    }

    stage('Commit & push changes') {
      steps {
        dir('manifests') {
          withCredentials([usernamePassword(credentialsId: "${GIT_CREDENTIALS}", usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
            sh """
              set -eux
              git config user.email "jenkins@local"
              git config user.name "Jenkins"
              git add "${TARGET_FILE}"
              git commit -m "chore: bump image to ${IMAGE_REPO}:${TAG_NAME}" || echo "No changes to commit"
              git push https://${GIT_USER}:${GIT_PASS}@${GIT_URL.replace('https://','')} HEAD:${GIT_BRANCH}
            """
          }
        }
      }
    }

    stage('(Optional) Dry-run diff against cluster') {
      when { expression { return false } } // set to true if you want this read-only check
      steps {
        dir('manifests') {
          sh 'kubectl diff -f "${TARGET_FILE}" || true'
        }
      }
    }
  }
}
