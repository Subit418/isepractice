pipeline {
    agent any

    environment {
        // change or remove if you want this configurable
        APP_PORT = '7070'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning repository from GitHub...'
                git url: 'https://github.com/Subit418/isepractice.git', branch: 'master'
            }
        }

        stage('Detect & Build') {
            steps {
                script {
                    if (fileExists('pom.xml')) {
                        echo "Maven project detected (pom.xml). Building with Maven..."
                        bat 'mvn -B clean package'
                    } else if (fileExists('gradlew') || fileExists('build.gradle')) {
                        echo "Gradle project detected. Building with Gradle..."
                        if (fileExists('gradlew')) {
                            // ensure gradlew is executable on non-Windows agents (harmless on Windows)
                            bat 'chmod +x gradlew || echo skipped chmod'
                            bat '.\\gradlew clean build'
                        } else {
                            bat 'gradle clean build'
                        }
                    } else if (fileExists('package.json')) {
                        echo "Node project detected (package.json). Installing dependencies and building..."
                        bat 'npm install'
                        // if package.json contains a build script, run it; harmless if it doesn't exist
                        bat 'npm run build || echo "no build script or build failed (continuing)"; exit 0'
                    } else {
                        echo "No recognized build file (pom.xml / build.gradle / package.json). Skipping automated build."
                    }
                }
            }
        }

        stage('Run Tests (if any)') {
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
                    // Archive common artifact locations
                    if (fileExists('target')) {
                        stash includes: 'target/**', name: 'maven-target' // optional stash
                        archiveArtifacts artifacts: 'target/**/*.jar, target/**/*.war', allowEmptyArchive: true
                        // junit for maven surefire
                        junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                    }
                    if (fileExists('build')) {
                        archiveArtifacts artifacts: 'build/libs/**/*.jar', allowEmptyArchive: true
                        junit allowEmptyResults: true, testResults: '**/build/test-results/**/*.xml'
                    }
                    if (fileExists('coverage') || fileExists('reports')) {
                        archiveArtifacts artifacts: 'coverage/**, reports/**', allowEmptyArchive: true
                    }
                    if (fileExists('package.json')) {
                        archiveArtifacts artifacts: 'dist/**, build/**', allowEmptyArchive: true
                    }
                }
            }
        }

        stage('Deploy / Run Application (agent-level)') {
            steps {
                script {
                    // Try to run built jar if present
                    if (fileExists('target') && bat(returnStatus: true, script: 'dir target\\*.jar > nul 2>&1') == 0) {
                        echo "Found jar in target/ — starting the jar (background)..."
                        // start jar in background using PowerShell Start-Process (Windows-friendly)
                        bat """
                        powershell -Command ^
                        \"$jar = Get-ChildItem -Path target -Filter *.jar | Select-Object -First 1; ^
                        if ($jar) { ^
                            Write-Host 'Starting:' $jar.FullName; ^
                            # kill any process using the desired port (best-effort)
                            try { $p = Get-NetTCPConnection -LocalPort ${APP_PORT} -ErrorAction SilentlyContinue ; if ($p) { $proc = Get-Process -Id $p.OwningProcess -ErrorAction SilentlyContinue; if ($proc) { $proc | Stop-Process -Force } } } catch { Write-Host 'port free/cleanup ignored' } ; ^
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
                        // On Windows, use start to run in background
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
        }
        failure {
            echo 'Pipeline failed!'
        }
        always {
            echo 'Pipeline finished (always block).'
        }
    }
}
