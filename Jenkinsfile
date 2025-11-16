pipeline {
 agent any
 environment {
 DEPLOY_DIR = 'C:\\inetpub\\wwwroot' // IIS web root folder
 }
 stages {
 stage('Checkout') {
 steps {
 echo 'Cloning project from GitHub...'
 git branch: 'main', url: 'https://github.com/repo/isepractice.git'
 }
 }
 stage('Build') {
 steps {
 echo 'Build Step: Check files in workspace'
 bat 'dir'
 }
 }
 stage('Deploy') {
 steps {
 echo "Deploying index.html to IIS folder"
 // Directly copy Home.html to webserver root
 bat "xcopy /Y index.html ${DEPLOY_DIR}\\"
 }
 }
 stage('Run HTTPS Server (Optional Test)') {
 steps {
 echo 'Skipping HTTPS server (use IIS instead)'
 // For testing, you can use: bat 'python -m http.server 8000'
 }
 }
 }
 post {
 success {
 echo 'Pipeline finished successfully! Visit: http://localhost/index.html'
 }
 failure {
 echo 'Pipeline failed! Check build logs.'
 }
 }
}

