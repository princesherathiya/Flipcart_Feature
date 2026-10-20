pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Building the website...'
                bat 'if exist index.html (exit /b 0) else (exit /b 1)'
                echo 'Build completed successfully!'
            }
        }
    }
}
