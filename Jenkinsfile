pipeline {
  agent any

  parameters {
    string(name: 'DOCKERTAG', defaultValue: 'latest', description: 'Tag from Pipeline 1')
  }

  environment {
    GIT_URL         = 'https://github.com/Kevinjosetom/kubernetesmanifest.git'
    GIT_BRANCH      = 'main'

    // Targets from your YAML
    DEPLOYMENT_NAME = 'flaskdemo'       // Deployment metadata.name
    CONTAINER_NAME  = 'flaskdemo'       // containers[].name
    IMAGE_REPO      = 'docker.io/kevinjosetom/myapp'
    TARGET_FILE     = 'deployment.yaml'

    // Jenkins credential (Username + Password/PAT)
    GITHUB_CREDS_ID = 'github'
  }

  stages {
    stage('Checkout manifests') {
      steps {
        dir('manifests') {
          // public clone (no creds needed to read)
          git branch: "${GIT_BRANCH}", url: "${GIT_URL}"
        }
      }
    }

    stage('Get yq (standalone)') {
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
          // Heredoc avoids all quoting issues and preserves the whole doc
          sh '''
            set -eux
            cat > prog.yq <<'YQ'
            if .kind=="Deployment" and .metadata.name==env(DEPLOYMENT_NAME) then
              (.spec.template.spec.containers[]
                | select(.name==env(CONTAINER_NAME))
                | .image) = env(IMAGE_REPO) + ":" + strenv(DOCKERTAG)
            else . end
            YQ

            ./yq e -i -f prog.yq "${TARGET_FILE}"
            ./yq e '.spec.template.spec.containers[] | {name, image}' "${TARGET_FILE}" || true
            git --no-pager diff -- "${TARGET_FILE}" || true
          '''
        }
      }
    }

    stage('Commit & push') {
      steps {
        dir('manifests') {
          withCredentials([usernamePassword(
            credentialsId: "${GITHUB_CREDS_ID}",
            usernameVariable: 'GIT_USER',
            passwordVariable: 'GIT_PASS'   // put your GitHub PAT in the password field
          )]) {
            sh '''
              set -eux
              git config user.email "jenkins@local"
              git config user.name  "Jenkins"
              git add "${TARGET_FILE}"
              git commit -m "chore: bump image to ${IMAGE_REPO}:${DOCKERTAG}" || echo "No changes to commit"

              # Build HTTPS remote with credentials and push to the same branch
              REMOTE="$(git config --get remote.origin.url | sed -E 's#^https?://##')"
              git push "https://${GIT_USER}:${GIT_PASS}@${REMOTE}" HEAD:"${GIT_BRANCH}"
            '''
          }
        }
      }
    }
  }

  post {
    success {
      echo "Updated ${TARGET_FILE} to ${IMAGE_REPO}:${DOCKERTAG} and pushed to ${GIT_BRANCH}."
    }
  }
}
