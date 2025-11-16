pipeline {
  agent any

  environment {
    // credentialsId must match the ID you set in Jenkins when adding username+password
    GIT_CREDENTIALS = 'github-pat'
    REPO_URL = 'https://github.com/Subit418/isepractice.git'
    // adjust TARGET_DIR to the folder where your static server will read files
    TARGET_DIR = "${env.WORKSPACE}/deployed-site" // default: deploy inside workspace (safe)
    // Example for system-wide serving: Linux: /var/www/html/portfolio or Windows: C:\\portfolio_site
    // TARGET_DIR = "/var/www/html/portfolio"
  }

  stages {
    stage('Checkout') {
      steps {
        // Use usernamePassword credentials for HTTPS checkout
        checkout([$class: 'GitSCM',
          branches: [[name: '*/master']],
          userRemoteConfigs: [[
            url: env.REPO_URL,
            credentialsId: env.GIT_CREDENTIALS
          ]]
        ])
      }
    }

    stage('Build (none)') {
      steps {
        echo "Static site — no build step."
      }
    }

    stage('Deploy') {
      steps {
        script {
          // create target dir
          sh '''
            set -e
            rm -rf "$TARGET_DIR"
            mkdir -p "$TARGET_DIR"
            cp -r ./* "$TARGET_DIR"/
            echo "Deployed to $TARGET_DIR"
          '''.trim()
        }
      }
      // For Windows agent, use bat instead:
      /*
      steps {
        bat """
        if exist "%TARGET_DIR%" rmdir /s /q "%TARGET_DIR%"
        mkdir "%TARGET_DIR%"
        xcopy /E /I /Y * "%TARGET_DIR%\\"
        echo Deployed to %TARGET_DIR%
        """
      }
      */
    }

    stage('Post deploy (status)') {
      steps {
        echo "Deployment finished. Serve the folder with a static server (see README)."
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: '**/*.html, **/*.css', fingerprint: true
    }
  }
}
