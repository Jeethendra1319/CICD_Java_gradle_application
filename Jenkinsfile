pipeline {
    agent {
        label 'ec2-fleet'
    }

    environment {
        aws_region = "us-east-1"
    }

    stages {
        stage('Checkout and Set Variables') {
            steps {
                script {
                    checkout scm
                    env.Docker_tag = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()

                    withCredentials([usernamePassword(credentialsId: 'aws-login-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                        def aws_account = sh(script: '''
                            export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                            export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                            aws sts get-caller-identity --query Account --output text
                        ''', returnStdout: true).trim()
                        env.aws_account_id = aws_account
                    }

                    echo "Docker tag: ${env.Docker_tag}"
                    echo "AWS Account ID: ${env.aws_account_id}"
                }
            }
        }

        stage('Initial Checks') {
            parallel {
                stage("Lint") {
                    steps {
                        script {
                            try {
                                sh 'chmod +x lint-all.sh'
                                sh './lint-all.sh'
                            } catch (err) {
                                currentBuild.result = 'UNSTABLE'
                                echo "Linting failed, please correct issues."
                            }
                        }
                    }
                }

                stage("Health Check") {
                    steps {
                        script {
                            try {
                                // Check if health-check.sh exists before executing it
                                sh '''
                                    if [ -f health-check.sh ]; then
                                        chmod +x health-check.sh
                                        ./health-check.sh
                                    else
                                        echo "health-check.sh not found, skipping."
                                    fi
                                '''
                            } catch (err) {
                                currentBuild.result = 'UNSTABLE'
                                echo "Health check failed."
                            }
                        }
                    }
                }
            }
        }

        stage('Build and Sonar Scan') {
            parallel {
                stage("Build") {
                    steps {
                        script {
                            docker.image('openjdk:11').inside('--user root') {
                                sh 'chmod +x gradlew'
                                sh './gradlew build'
                            }
                        }
                    }
                }

                stage("Sonar Scan") {
                    steps {
                        script {
                            try {
                                docker.image('openjdk:17').inside('--user root') {
                                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                                        sh 'chmod +x gradlew'
                                        sh './gradlew sonarqube'
                                    }

                                    timeout(time: 1, unit: 'HOURS') {
                                        def qg = waitForQualityGate()
                                        if (qg.status != 'OK') {
                                            error "Pipeline aborted due to Sonar Quality Gate failure: ${qg.status}"
                                        }
                                    }
                                }
                            } catch (err) {
                                currentBuild.result = 'UNSTABLE'
                                echo "SonarQube scan failed: ${err.getMessage()}"
                            }
                        }
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    sh "docker build -t spring-app:${env.Docker_tag} ."
                    currentBuild.description = "spring-app:${env.Docker_tag}"
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    try {
                        withCredentials([usernamePassword(credentialsId: 'aws-login-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                            sh """
                                aws ecr get-login-password --region ${env.aws_region} | docker login --username AWS --password-stdin ${env.aws_account_id}.dkr.ecr.${env.aws_region}.amazonaws.com
                                docker tag spring-app:${env.Docker_tag} ${env.aws_account_id}.dkr.ecr.${env.aws_region}.amazonaws.com/spring-app:${env.Docker_tag}
                                docker push ${env.aws_account_id}.dkr.ecr.${env.aws_region}.amazonaws.com/spring-app:${env.Docker_tag}
                                docker rmi ${env.aws_account_id}.dkr.ecr.${env.aws_region}.amazonaws.com/spring-app:${env.Docker_tag} spring-app:${env.Docker_tag}
                            """
                        }
                    } catch (err) {
                        currentBuild.result = 'UNSTABLE'
                        echo "Docker Push failed: ${err.getMessage()}"
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
