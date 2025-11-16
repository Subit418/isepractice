pipeline {
    agent any

    environment {
        APP_PORT = '7070'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning repository from GitHub...'
                git url: 'https://github.com/Subit418/isepractice.git', branch: 'master'
            }
        }

        stage('Detect Project Type') {
            steps {
                script {
                    boolean hasIndex = fileExists('index.html')
                    boolean hasAnyHtml = false
                    // quick check for any .html files
                    def htmlFiles = findFiles(glob: '**/*.html')
                    if (htmlFiles?.length > 0) {
                        hasAnyHtml = true
                    }
                    if (hasIndex || hasAnyHtml) {
                        echo "Static site detected (HTML files present). Will serve static content."
                        currentBuild.description = "Static site"
                        env.IS_STATIC = "true"
                    } else {
                        echo "No HTML files detected. Proceeding with normal build detections."
                        env.IS_STATIC = "false"
                    }
                }
            }
        }

        stage('Detect & Build (if non-static)') {
            when {
                expression { return env.IS_STATIC == "false" }
            }
            steps {
                script {
                    if (fileExists('pom.xml')) {
                        echo "Maven project detected (pom.xml). Building with Maven..."
                        bat 'mvn -B clean package'
                    } else if (fileExists('gradlew') || fileExists('build.gradle')) {
                        echo "Gradle project detected. Building with Gradle..."
                        if (fileExists('gradlew')) {
                            bat 'chmod +x gradlew || echo skipped chmod'
                            bat '.\\gradlew clean build'
                        } else {
                            bat 'gradle clean build'
                        }
                    } else if (fileExists('package.json')) {
                        echo "Node project detected (package.json). Installing dependencies and building..."
                        bat 'npm install'
                        bat 'npm run build || echo "no build script or build failed (continuing)"; exit 0'
                    } else {
                        echo "No recognized build file (pom.xml / build.gradle / package.json). Skipping automated build."
                    }
                }
            }
        }

        stage('Run Tests (if any)') {
            when {
                expression { return env.IS_STATIC == "false" }
            }
            steps {
                script {
                    if (fileExists('pom.xml')) {
                        echo "Running Maven tests..."
                        bat 'mvn test'
                    } else if (fileExists('build.gradle') || fileExists('gradlew')) {
                        echo "Running Gradle tests..."
                        if (fileExists('gradlew')) {
                            bat '.\\gradlew test'
                        } else {
                            bat 'gradle test'
                        }
                    } else if (fileExists('package.json')) {
                        echo "Running npm test (if defined)..."
                        bat 'npm test || echo "npm test returned non-zero or not defined; continuing"'
                    } else {
                        echo "No test step for unknown project type."
                    }
                }
            }
        }

        stage('Archive Artifacts & Test Reports') {
            steps {
                script {
                    if (fileExists('target')) {
                        archiveArtifacts artifacts: 'target/**/*.jar, target/**/*.war', allowEmptyArchive: true
                        junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                    }
                    if (fileExists('build')) {
                        archiveArtifacts artifacts: 'build/libs/**/*.jar', allowEmptyArchive: true
                        junit allowEmptyResults: true, testResults: '**/build/test-results/**/*.xml'
                    }
                    if (fileExists('package.json')) {
                        archiveArtifacts artifacts: 'dist/**, build/**', allowEmptyArchive: true
                    }
                }
            }
        }

        stage('Deploy / Serve Static Site') {
            when {
                expression { return env.IS_STATIC == "true" }
            }
            steps {
                script {
                    echo "Preparing to serve static site on port ${APP_PORT}..."

                    // best-effort: kill any process using the port (Windows PowerShell)
                    bat """
                    powershell -Command ^
                    "try { ^
                        $conns = Get-NetTCPConnection -LocalPort ${APP_PORT} -ErrorAction SilentlyContinue; ^
                        if ($conns) { ^
                            $procs = $conns | ForEach-Object { $_.OwningProcess } | Sort-Object -Unique; ^
                            foreach ($pid in $procs) { ^
                                try { Stop-Process -Id $pid -Force -ErrorAction SilentlyContinue; Write-Host 'Stopped process' $pid 'on port ${APP_PORT}'; } catch { Write-Host 'Failed to stop process' $pid } ^
                            } ^
                        } else { Write-Host 'No process found on port ${APP_PORT}'; } ^
                    } catch { Write-Host 'Port cleanup ignored: ' $_ }"
                    """

                    // Start server: prefer Python, else fallback to npx http-server
                    // Start-Process will run it in background and write output to serve.log
                    // Use powershell Start-Process to keep it running after the step finishes
                    bat """
                    powershell -Command ^
                    "if (Get-Command python -ErrorAction SilentlyContinue) { ^
                        Write-Host 'Python found. Starting python -m http.server ${APP_PORT}'; ^
                        Start-Process -FilePath python -ArgumentList '-m','http.server','${APP_PORT}' -WorkingDirectory '${WORKSPACE}' -RedirectStandardOutput '${WORKSPACE}\\serve.log' -RedirectStandardError '${WORKSPACE}\\serve.log' -NoNewWindow -WindowStyle Hidden; ^
                        Write-Host 'Started python http.server (background). Logs -> ${WORKSPACE}\\\\serve.log' ^
                    } elseif (Get-Command npx -ErrorAction SilentlyContinue) { ^
                        Write-Host 'npx found. Starting http-server via npx on port ${APP_PORT}'; ^
                        Start-Process -FilePath npx -ArgumentList 'http-server','-p','${APP_PORT}','.' -WorkingDirectory '${WORKSPACE}' -RedirectStandardOutput '${WORKSPACE}\\serve.log' -RedirectStandardError '${WORKSPACE}\\serve.log' -NoNewWindow -WindowStyle Hidden; ^
                        Write-Host 'Started npx http-server (background). Logs -> ${WORKSPACE}\\\\serve.log' ^
                    } else { ^
                        Write-Host 'Neither python nor npx available on agent. Cannot auto-serve static site. Please install Python or Node (with http-server).'; ^
                        Exit 1 ^
                    }"
                    """
                    echo "Static server launched. Open: http://<JENKINS-IP>:${APP_PORT} (or http://localhost:${APP_PORT} if Jenkins runs locally)"
                }
            }
        }

        stage('Deploy / Run Application (fallback)') {
            when {
                expression { return env.IS_STATIC == "false" }
            }
            steps {
                script {
                    // Existing fallback behavior for non-static apps (try to start jars or node start)
                    if (fileExists('target') && bat(returnStatus: true, script: 'dir target\\*.jar > nul 2>&1') == 0) {
                        echo "Found jar in target/ — starting the jar (background)..."
                        bat """
                        powershell -Command ^
                        \"$jar = Get-ChildItem -Path target -Filter *.jar | Select-Object -First 1; ^
                        if ($jar) { ^
                            Write-Host 'Starting:' $jar.FullName; ^
                            try { $p = Get-NetTCPConnection -LocalPort ${APP_PORT} -ErrorAction SilentlyContinue ; if ($p) { $proc = Get-Process -Id $p.OwningProcess -ErrorAction SilentlyContinue; if ($proc) { $proc | Stop-Process -Force } } } catch { Write-Host 'port cleanup ignored' } ; ^
                            Start-Process -FilePath 'java' -ArgumentList '-jar', $jar.FullName -NoNewWindow -PassThru | Out-Null; ^
                            Write-Host 'Application started (check logs on agent).' ^
                        } else { Write-Host 'No jar found to run.' }\" 
                        """
                    } else if (fileExists('build\\libs') && bat(returnStatus: true, script: 'dir build\\libs\\*.jar > nul 2>&1') == 0) {
                        echo "Found jar in build/libs — starting the jar (background)..."
                        bat """
                        powershell -Command ^
                        \"$jar = Get-ChildItem -Path build\\libs -Filter *.jar | Select-Object -First 1; ^
                        if ($jar) { Start-Process -FilePath 'java' -ArgumentList '-jar', $jar.FullName -NoNewWindow -PassThru | Out-Null; Write-Host 'Started jar.' } else { Write-Host 'No jar found.' }\" 
                        """
                    } else if (fileExists('package.json') && (bat(returnStatus: true, script: 'type package.json | findstr /I /C:"\\"start\\""' ) == 0 || bat(returnStatus: true, script: 'type package.json | findstr /I /C:"serve"') == 0)) {
                        echo "Starting Node app (npm start) in background..."
                        bat 'start /B cmd /c "npm start > app.log 2>&1"'
                        echo "npm start launched (output -> app.log)"
                    } else {
                        echo "No known deployable artifact found. Skipping auto-deploy."
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
            echo "If a static site was detected, open: http://<JENKINS-IP>:${APP_PORT}"
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            echo 'Pipeline finished (always block).'
        }
    }
}
