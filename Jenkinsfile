// Jenkinsfile - Platform-aware (runs on Linux or Windows agents)
// Assumptions:
// - SSH key credential ID (SSH Username with private key): deploy-ssh-key (for ssh deploy)
// - GitHub credentials (Username with password): GITHUB_CREDS (username=GH user, password=PAT)
// - If running SSH deploy, prefer Linux agent (rsync + ssh). On Windows, SSH deploy will error unless agent has rsync/ssh tools.

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
    SSH_CREDENTIALS_ID = 'deploy-ssh-key'
    GITHUB_CREDENTIALS_ID = 'github-pat'
    TMPDIR = "${env.WORKSPACE}\\_deploy_tmp" // will be adapted for Unix when using commands
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '20'))
    timestamps()
  }

  // helper method to run commands depending on agent OS
  // NOTE: we implement as scripted within steps { script { ... } } below
  stages {
    stage('Checkout') {
      steps {
        echo "Checking out source..."
        checkout scm
      }
    }

    stage('Prepare') {
      steps {
        script {
          if (isUnix()) {
            env.TMPDIR = "${env.WORKSPACE}/_deploy_tmp"
            sh """
              rm -rf "${env.TMPDIR}"
              mkdir -p "${env.TMPDIR}"
              # copy workspace to temp (exclude .git)
              rsync -a --exclude='.git' ./ "${env.TMPDIR}/"
              ls -la "${env.TMPDIR}"
            """
          } else {
            // Windows agent - use robocopy for copying (robocopy returns non-zero on some cases so ignore errors)
            env.TMPDIR = "${env.WORKSPACE}\\\\_deploy_tmp"
            bat """
              if exist "%TMPDIR%" rmdir /S /Q "%TMPDIR%"
              mkdir "%TMPDIR%"
              REM copy files excluding .git folder
              powershell -Command "Get-ChildItem -Path . -Exclude '.git' -Force | ForEach-Object { \$dest='${env.TMPDIR}\\\\' + \$_.Name; if (\$_.PSIsContainer) { robocopy \$_.FullName \$dest /E } else { Copy-Item \$_.FullName \$dest } }"
              dir "%TMPDIR%"
            """
          }
        }
      }
    }

    stage('Build (optional)') {
      steps {
        echo "No build step for static site by default. Add build/minify steps if needed."
      }
    }

    stage('Deploy') {
      when { expression { params.DEPLOY_METHOD != 'none' } }
      steps {
        script {
          if (params.DEPLOY_METHOD == 'ssh') {
            if (!isUnix()) {
              error "SSH/rsync deploy selected but agent is Windows. Either run on a Linux agent that has rsync/ssh, or install Unix tools (rsync/ssh) on this Windows node."
            }
            echo "Deploy via SSH (rsync) to ${params.REMOTE_USER}@${params.REMOTE_HOST}:${params.REMOTE_PATH}"
            sshagent (credentials: [env.SSH_CREDENTIALS_ID]) {
              sh """
                set -e
                rsync -av --delete --exclude='.git' "${env.TMPDIR}/" ${params.REMOTE_USER}@${params.REMOTE_HOST}:"${params.REMOTE_PATH}"
                echo "SSH deploy complete."
              """
            }
          }
          else if (params.DEPLOY_METHOD == 'gh-pages') {
            echo "Deploy to GitHub Pages for ${params.GITHUB_USERNAME}/${params.REPO_NAME}"
            withCredentials([usernamePassword(credentialsId: env.GITHUB_CREDENTIALS_ID, usernameVariable: 'GH_USER', passwordVariable: 'GH_TOKEN')]) {
              if (isUnix()) {
                sh """
                  set -e
                  cd "${env.TMPDIR}"
                  git init
                  # create or switch to gh-pages
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
                  echo "gh-pages branch updated."
                """
              } else {
                // Windows bat commands - use Groovy string interpolation for params and credentials
                def forceFlag = params.FORCE_PUSH_GH_PAGES ? "true" : "false"
                bat """
                  cd /d "${env.TMPDIR}"
                  git init
                  powershell -Command "if (git show-ref --verify --quiet refs/heads/gh-pages) { git checkout gh-pages } else { git checkout -b gh-pages }"
                  git config user.email "jenkins@ci.local"
                  git config user.name "Jenkins CI"
                  git add -A
                  powershell -Command "if ((git diff --cached --quiet) -eq $true) { Write-Host 'No changes to deploy to gh-pages.' } else { git commit -m 'Deploy to gh-pages from Jenkins build ${env.BUILD_NUMBER}' }"
                  set REMOTE_URL=https://%GH_USER%:%GH_TOKEN%@github.com/${params.GITHUB_USERNAME}/${params.REPO_NAME}.git
                  ${ forceFlag == "true" ? "git push --force \"%REMOTE_URL%\" gh-pages:gh-pages" : "git push \"%REMOTE_URL%\" gh-pages:gh-pages" }
                  echo gh-pages branch updated
                """
              } // isUnix() check end
            } // withCredentials end
          } else {
            error "Unknown DEPLOY_METHOD: ${params.DEPLOY_METHOD}"
          }
        } // script end
      } // steps end
    } // Deploy stage end
  } // stages end

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
    failure {
      echo "Build or deployment FAILED. See console output."
    }
    always {
      script {
        if (isUnix()) {
          sh "rm -rf \"${env.TMPDIR}\" || true"
        } else {
          bat "rmdir /S /Q \"${env.TMPDIR}\" || exit 0"
        }
      }
    }
  }
}
