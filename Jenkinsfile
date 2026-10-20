
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
                sh 'test -f index.html'
                echo 'Build completed successfully!'
            }
        }
    }
}
