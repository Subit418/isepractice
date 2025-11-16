// Jenkinsfile - Combined SSH deploy and GitHub Pages deploy (uses username/password Jenkins creds for GitHub PAT)
// Pre-reqs:
//  - SSH key credential (kind: SSH Username with private key) with ID: deploy-ssh-key
//  - GitHub credential (kind: Username with password) with ID: GITHUB_CREDS
//    -> Username: your GitHub username
//    -> Password: your classic PAT

pipeline {
  agent any

  parameters {
    choice(name: 'DEPLOY_METHOD', choices: ['none', 'ssh', 'gh-pages'], description: 'Choose deploy method')
    string(name: 'REMOTE_HOST', defaultValue: 'your.server.com', description: 'Remote SSH host (for ssh deploy)')
    string(name: 'REMOTE_USER', defaultValue: 'deployuser', description: 'Remote SSH user (for ssh deploy)')
    string(name: 'REMOTE_PATH', defaultValue: '/var/www/portfolio', description: 'Remote path to deploy to (for ssh deploy)')
    string(name: 'GITHUB_USERNAME', defaultValue: '<USERNAME>', description: 'GitHub username/org (for gh-pages deploy)')
    string(name: 'REPO_NAME', defaultValue: 'portfolio', description: 'GitHub repository name (for gh-pages deploy)')
    booleanParam(name: 'FORCE_PUSH_GH_PAGES', defaultValue: true, description: 'Force push gh-pages branch (overwrites remote gh-pages)')
  }

  environment {
    SSH_CREDENTIALS_ID = 'deploy-ssh-key'     // change if your SSH credential id is different
    GITHUB_CREDENTIALS_ID = 'github-pat'    // change to the Jenkins credentials id you created (username/password)
    TMPDIR = "${env.WORKSPACE}/_deploy_tmp"
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '20'))
    timestamps()
  }

  stages {
    stage('Checkout') {
      steps {
        echo "Checking out source..."
        checkout scm
      }
    }

    stage('Prepare') {
      steps {
        echo "Preparing deployment directory..."
        sh '''
          rm -rf "${TMPDIR}"
          mkdir -p "${TMPDIR}"
          rsync -a --exclude='.git' ./ "${TMPDIR}/"
          ls -la "${TMPDIR}"
        '''
      }
    }

    stage('Build (optional)') {
      steps {
        echo "No build step by default for static site."
      }
    }

    stage('Deploy') {
      when { expression { params.DEPLOY_METHOD != 'none' } }
      steps {
        script {
          if (params.DEPLOY_METHOD == 'ssh') {
            echo "Deploying via SSH to ${params.REMOTE_USER}@${params.REMOTE_HOST}:${params.REMOTE_PATH}"
            sshagent (credentials: [env.SSH_CREDENTIALS_ID]) {
              sh '''
                set -e
                rsync -av --delete --exclude='.git' "${TMPDIR}/" ${REMOTE_USER}@${REMOTE_HOST}:"${REMOTE_PATH}"
                echo "SSH deploy complete."
              '''
            }
          } else if (params.DEPLOY_METHOD == 'gh-pages') {
            echo "Deploying to GitHub Pages for ${params.GITHUB_USERNAME}/${params.REPO_NAME}"
            // Use username/password credentials (username = GitHub username, password = classic PAT)
            withCredentials([usernamePassword(credentialsId: env.GITHUB_CREDENTIALS_ID, usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
              sh '''
                set -e
                cd "${TMPDIR}"

                git init
                # create or switch to gh-pages branch
                if git show-ref --verify --quiet refs/heads/gh-pages; then
                  git checkout gh-pages
                else
                  git checkout -b gh-pages
                fi

                git config user.email "jenkins@ci.local"
                git config user.name "Jenkins CI"

                git add -A

                if git diff --cached --quiet; then
                  echo "No changes to deploy to gh-pages."
                else
                  git commit -m "Deploy to gh-pages from Jenkins build ${env.BUILD_NUMBER}"
                fi

                REMOTE_URL="https://${GH_USER}:${GH_TOKEN}@github.com/${params.GITHUB_USERNAME}/${params.REPO_NAME}.git"
                if [ "${params.FORCE_PUSH_GH_PAGES}" = "true" ]; then
                  git push --force "$REMOTE_URL" gh-pages:gh-pages
                else
                  git push "$REMOTE_URL" gh-pages:gh-pages
                fi

                echo "gh-pages updated."
              '''
            } // withCredentials
          } else {
            error "Unknown DEPLOY_METHOD: ${params.DEPLOY_METHOD}"
          }
        } // script
      } // steps
    } // stage Deploy
  } // stages

  post {
    success {
      script {
        if (params.DEPLOY_METHOD == 'ssh') {
          echo "Deployment SUCCESS via SSH."
        } else if (params.DEPLOY_METHOD == 'gh-pages') {
          echo "Deployment SUCCESS to https://${params.GITHUB_USERNAME}.github.io/${params.REPO_NAME}/"
        } else {
          echo "Build SUCCESS (no deploy)."
        }
      }
    }
    failure { echo "Build or deployment FAILED." }
    always {
      echo "Cleaning up..."
      sh 'rm -rf "${TMPDIR}" || true'
    }
  }
}
