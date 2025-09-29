pipeline {
  agent any

  parameters {
    string(name: 'DOCKERTAG', defaultValue: 'latest', description: 'Tag from Pipeline 1')
  }

  environment {
    GIT_URL         = 'https://github.com/Kevinjosetom/kubernetesmanifest.git'
    GIT_BRANCH      = 'main'
    DEPLOYMENT_NAME = 'flaskdemo'
    CONTAINER_NAME  = 'flaskdemo'      // change if needed
    IMAGE_REPO      = 'docker.io/kevinjosetom/myapp'
    TARGET_FILE     = 'deployment.yaml'
    GITHUB_TOKEN_ID = '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'
  }

  stages {
    stage('Checkout manifests') {
      steps {
        dir('manifests') { git branch: "${GIT_BRANCH}", url: "${GIT_URL}" }
      }
    }

    stage('Get yq & patch') {
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
          sh """
            set -eux
            ./yq -i '
              if .kind=="Deployment" and .metadata.name=="${DEPLOYMENT_NAME}" then
                (.spec.template.spec.containers[]
                  | select(.name=="${CONTAINER_NAME}")
                  | .image) = "${IMAGE_REPO}:" + strenv(DOCKERTAG)
              else . end
            ' "${TARGET_FILE}"
            git --no-pager diff -- "${TARGET_FILE}" || true
          """
        }
      }
    }

    stage('Commit & push') {
      steps {
        dir('manifests') {
          withCredentials([string(credentialsId: "${GITHUB_TOKEN_ID}", variable: 'GH_TOKEN')]) {
            sh '''
              set -eux
              git config user.email "kevin.tom@velocis.co.in"
              git config user.name  "Kevinjosetom"
              git add "${TARGET_FILE}"
              git commit -m "chore: bump image to docker.io/kevinjosetom/myapp:${DOCKERTAG}" || echo "No changes to commit"
              REMOTE="$(git config --get remote.origin.url | sed -E 's#https?://##')"
              git push "https://${GH_TOKEN}@${REMOTE}" HEAD:'"$GIT_BRANCH"'
            '''
          }
        }
      }
    }
  }
}
