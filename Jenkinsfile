pipeline {
  agent any

  parameters {
    string(name: 'DOCKERTAG', defaultValue: 'latest', description: 'Tag from Pipeline 1')
  }

  environment {
    GIT_URL         = 'https://github.com/Kevinjosetom/kubernetesmanifest.git'  // <-- your repo
    GIT_BRANCH      = 'main'

    DEPLOYMENT_NAME = 'flaskdemo'
    CONTAINER_NAME  = 'myapp'
    IMAGE_REPO      = 'docker.io/kevinjosetom/myapp'     // <-- your image repo
    TARGET_FILE     = 'deployment.yaml'                  // change if path differs
  }

  stages {
    stage('Checkout manifests') {
      steps {
        dir('manifests') {
          git branch: "${GIT_BRANCH}", url: "${GIT_URL}"
        }
      }
    }

    stage('Get yq (standalone)') {     // avoids snap issues under /var/lib/jenkins
      steps {
        dir('manifests') {
          sh '''
            set -eux
            case "$(uname -m)" in
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
            echo "Using DOCKERTAG=${DOCKERTAG}"
            ./yq -i '
              select(.kind=="Deployment" and .metadata.name=="${DEPLOYMENT_NAME}") |
              .spec.template.spec.containers[] |
              select(.name=="${CONTAINER_NAME}") |
              .image = "${IMAGE_REPO}:" + strenv(DOCKERTAG)
            ' "${TARGET_FILE}"
            git --no-pager diff -- "${TARGET_FILE}" || true
          """
        }
      }
    }

    stage('Commit & push') {
      steps {
        dir('manifests') {
            sh '''
              set -eux
              git config user.email "jenkins@local"
              git config user.name  "Jenkins"
              git add "${TARGET_FILE}"
              git commit -m "chore: bump image to '"${IMAGE_REPO}"':'"${DOCKERTAG}"'" || echo "No changes to commit"
              REMOTE="$(git config --get remote.origin.url | sed -E 's#https?://##')"
              git push "https://${GIT_USER}:${GIT_PASS}@${REMOTE}" HEAD:'"${GIT_BRANCH}"'
            '''
        }
      }
    }
  }
}
