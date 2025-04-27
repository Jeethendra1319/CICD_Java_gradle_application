pipeline {
    agent { label 'ec2-fleet' }  // Just use your EC2 agent normally

    stages {
        stage('build') {
            steps {
                script {
                    sh 'chmod +x gradlew'
                    sh './gradlew build'
                }
            }
        }
        stage('docker build') {
            steps {
                script {
                    // Here, Docker must be installed already on EC2
                    sh 'docker build -t my-app .'
                    sh 'docker images'
                }
            }
        }
    }
}
