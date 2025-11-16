pipeline {
  agent any

  environment {
    DEPLOY_DIR = 'C:\\inetpub\\wwwroot'    // IIS web root folder
    GIT_CREDS  = 'github-pat'           // your credentials ID in Jenkins
  }

  stages {
    stage('Checkout') {
      steps {
        echo "Cloning project from GitHub with credentialsId=${env.GIT_CREDS}"
        git branch: 'master',
            url: 'https://github.com/Subit418/isepractice.git',
            credentialsId: "${env.GIT_CREDS}"
      }
    }

    stage('Build - list workspace') {
      steps {
        bat """
          echo ===== Workspace: %WORKSPACE% =====
          echo User: %USERNAME%
          echo ---- Root files ----
          dir "%WORKSPACE%" /A
          echo ---- assets (if present) ----
          if exist "%WORKSPACE%\\assets" dir "%WORKSPACE%\\assets"
          echo ---- deploy.ps1 presence ----
          if exist "%WORKSPACE%\\deploy.ps1" ( echo deploy.ps1 FOUND ) else ( echo deploy.ps1 NOT FOUND )
        """
      }
    }

    stage('Deploy') {
      steps {
        bat """
          echo === Deploy stage ===
          setlocal

          rem If a deploy.ps1 exists, run it (preferred).
          if exist "%WORKSPACE%\\deploy.ps1" (
            echo Found deploy.ps1 - invoking PowerShell
            powershell -NoProfile -ExecutionPolicy Bypass -File "%WORKSPACE%\\deploy.ps1" -Workspace "%WORKSPACE%" -Dest "${DEPLOY_DIR}"
            if errorlevel 1 (
              echo ERROR: deploy.ps1 failed
              exit /b 2
            )
          ) else (
            echo deploy.ps1 not found - using fallback batch copy
            rem ensure destination exists
            if not exist "${DEPLOY_DIR}" (
              echo Destination ${DEPLOY_DIR} does not exist. Attempting to create.
              mkdir "${DEPLOY_DIR}"
              if errorlevel 1 (
                echo ERROR: Unable to create destination ${DEPLOY_DIR}
                exit /b 5
              )
            )

            set COPY_ERROR=0

            rem copy index.html
            if exist "%WORKSPACE%\\index.html" (
              echo Copying index.html -> ${DEPLOY_DIR}
              xcopy /Y /Q "%WORKSPACE%\\index.html" "${DEPLOY_DIR}\\"
              if errorlevel 1 set COPY_ERROR=1
            ) else (
              echo index.html not found in workspace
            )

            rem copy about.html
            if exist "%WORKSPACE%\\about.html" (
              echo Copying destination.html -> ${DEPLOY_DIR}
              xcopy /Y /Q "%WORKSPACE%\\about.html" "${DEPLOY_DIR}\\"
              if errorlevel 1 set COPY_ERROR=1
            ) else (
              echo about.html not found in workspace
            )

            rem copy assets folder recursively if present
            if exist "%WORKSPACE%\\assets" (
              echo Copying assets folder -> ${DEPLOY_DIR}\\assets
              xcopy /E /I /Y /Q "%WORKSPACE%\\assets" "${DEPLOY_DIR}\\assets\\"
              if errorlevel 1 set COPY_ERROR=1
            ) else (
              echo assets folder not present
            )

            rem fail if nothing was copied
            if not exist "%WORKSPACE%\\index.html" if not exist "%WORKSPACE%\\about.html" if not exist "%WORKSPACE%\\assets" (
              echo ERROR: No deployable artifacts found (index.html or about.html or assets/)
              dir "%WORKSPACE%"
              exit /b 4
            )

            if "%COPY_ERROR%"=="1" (
              echo ERROR: one or more copy operations failed - check permissions for ${DEPLOY_DIR}
              exit /b 3
            )
          )

          echo === Deploy stage finished ===
        """
      }
    }

    stage('Verify HTTP (server-side)') {
      steps {
        bat """
          echo Verifying http://localhost/index.html on Jenkins host
          powershell -NoProfile -Command ^
            "try { \$r = Invoke-WebRequest -UseBasicParsing -Uri 'http://localhost/index.html' -TimeoutSec 10; Write-Output 'STATUS:' \$r.StatusCode; if (\$r.StatusCode -eq 200) { Write-Output 'Snippet:'; Write-Output \$r.Content.Substring(0,[Math]::Min(300,\$r.Content.Length)) } else { Write-Output 'Non-200 status returned'; exit 6 } } catch { Write-Output 'HTTP check failed:'; Write-Output \$_.Exception.Message; exit 6 }"
        """
      }
    }
  }

  post {
    success {
      echo "Pipeline finished successfully. Visit: http://<jenkins-host-or-ip>/index.html"
    }
    failure {
      echo "Pipeline failed — check the console log above. Common issues: credentials mismatch for checkout, missing files in workspace, or no write permission to ${DEPLOY_DIR}."
    }
  }
}
