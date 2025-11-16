pipeline {
    agent any

    environment {
        APP_PORT = '7070'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Pulling website from GitHub...'
                git url: 'https://github.com/Subit418/isepractice.git', branch: 'master'
            }
        }

        stage('Start Static Server') {
            steps {
                echo "Starting static website on port ${APP_PORT}..."

                // Kill old process on that port (Windows only)
                bat """
                powershell -Command ^
                "try { ^
                    Get-NetTCPConnection -LocalPort ${APP_PORT} -ErrorAction SilentlyContinue | ^
                    ForEach-Object { Stop-Process -Id $_.OwningProcess -Force -ErrorAction SilentlyContinue } ^
                } catch {}"
                """

                // Start python server in background
                bat """
                powershell -Command ^
                "Start-Process python -ArgumentList '-m','http.server','${APP_PORT}' -WorkingDirectory '${WORKSPACE}' -WindowStyle Hidden"
                """

                echo "Server started! Open: http://<JENKINS-IP>:${APP_PORT}"
            }
        }
    }

    post {
        success {
            echo "Website is running at: http://<JENKINS-IP>:${APP_PORT}"
        }
        failure {
            echo "Pipeline failed."
        }
    }
}
