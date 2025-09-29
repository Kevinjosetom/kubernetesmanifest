pipeline {
  agent any

  parameters {
    string(name: 'TAG_NAME', defaultValue: 'latest', description: 'Image tag from Pipeline 1')
  }

  environment {
    GIT_CREDENTIALS = 'github-token-or-userpass'  // Jenkins credential id
    GIT_URL         = 'https://github.com/Kevinjosetom/kubernetesmanifest.git'
    GIT_BRANCH      = 'main'  // change if different
    // target deployment identifiers:
    DEPLOYMENT_NAME = 'flaskdemo'
    CONTAINER_NAME  = 'myapp'
    IMAGE_REPO      = 'docker.io/kevinjosetom/myapp'
  }

  stages {
    stage('Checkout manifests repo') {
      steps {
        dir('manifests') {
          // Use the Git plugin so Jenkins manages remotes/branches
          git branch: "${GIT_BRANCH}", url: "${GIT_URL}", credentialsId: "${GIT_CREDENTIALS}"
        }
      }
    }

    stage('Patch deployment YAML (file only)') {
      steps {
        dir('manifests') {
          // Ensure yq is present on the agent
          sh 'yq --version'
          // Edit the specific file; update the path if different
          sh """
            echo "Using TAG_NAME=${TAG_NAME}"
            yq -i '
              select(.kind=="Deployment" and .metadata.name=="${DEPLOYMENT_NAME}") |
              .spec.template.spec.containers[] |
              select(.name=="${CONTAINER_NAME}") |
              .image = "${IMAGE_REPO}:" + (env.TAG_NAME)
            ' deployment.yaml
          """
          // Optional: show diff of file changes locally
          sh 'git --no-pager diff -- deployment.yaml || true'
        }
      }
    }

    stage('Commit & push changes') {
      steps {
        dir('manifests') {
          withCredentials([usernamePassword(credentialsId: "${GIT_CREDENTIALS}", usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
            sh """
              git config user.email "jenkins@local"
              git config user.name "Jenkins"
              git add deployment.yaml
              git commit -m "chore: bump image to ${IMAGE_REPO}:${TAG_NAME}" || echo "No changes to commit"
              # Push back to the same branch
              git push https://${GIT_USER}:${GIT_PASS}@${GIT_URL.replace('https://','')} HEAD:${GIT_BRANCH}
            """
          }
        }
      }
    }

    stage('(Optional) Dry-run diff against cluster') {
      when { expression { return false } } // flip to true if you want this
      steps {
        dir('manifests') {
          sh 'kubectl diff -f deployment.yaml || true'  // read-only
        }
      }
    }
  }
}
