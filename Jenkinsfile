pipeline {
    agent any

    environment {
        APP_PORT = '7070'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning website repository...'
                git url: 'https://github.com/Subit418/isepractice.git', branch: 'master'
            }
        }

        stage('Start Static Server') {
            steps {
                echo "Preparing to serve static site on port %APP_PORT%..."

                // Best-effort: stop any process listening on the port (Windows)
                // then start python -m http.server in the background and redirect logs to serve.log
                bat """
                echo Stopping any process listening on port %APP_PORT% (if present)...
                for /f "tokens=5" %%a in ('netstat -ano ^| findstr :%APP_PORT% ^| findstr LISTENING') do (
                  echo Killing PID %%a
                  taskkill /PID %%a /F > nul 2>&1 || echo failed to kill %%a
                )

                echo Starting python HTTP server in background (logs -> serve.log)...
                start "" /B cmd /c "python -m http.server %APP_PORT% > serve.log 2>&1"
                echo Server started (background). Check serve.log in workspace for output.
                """
            }
        }
    }

    post {
        success {
            echo "If Jenkins runs locally open: http://localhost:%APP_PORT%"
            echo "If Jenkins runs on another machine open: http://<JENKINS-IP>:%APP_PORT%"
        }
        failure {
            echo "Pipeline failed. Check console output."
        }
    }
}
