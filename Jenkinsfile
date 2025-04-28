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
                    env.aws_account_id = sh(script: 'aws sts get-caller-identity --query Account --output text', returnStdout: true).trim()
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
                                return
                            }
                        }
                    }
                }

                stage("Health Check") {
                    steps {
                        script {
                            try {
                                sh 'chmod +x health-check.sh'
                                sh './health-check.sh'
                            } catch (err) {
                                currentBuild.result = 'UNSTABLE'
                                echo "Health check failed."
                                return
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
                            docker.image('openjdk:11').inside('--user root') {
                                try {
                                    withSonarQubeEnv(credentialsId: 'sonar-token') {
                                        sh 'chmod +x gradlew'
                                        sh './gradlew sonarqube'
                                    }
                                } catch (err) {
                                    currentBuild.result = 'UNSTABLE'
                                    echo "SonarQube scan failed: ${err}"
                                    return
                                }

                                timeout(time: 1, unit: 'HOURS') {
                                    def qg = waitForQualityGate()
                                    if (qg.status != 'OK') {
                                        error "Pipeline aborted due to Sonar Quality Gate failure: ${qg.status}"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    sh "docker build -t spring-app:${Docker_tag} ."
                    currentBuild.description = "spring-app:${Docker_tag}"
                }
            }
        }

        stage('Docker Push') {
            steps {
                script {
                    sh '''
                        aws ecr get-login-password --region ${aws_region} | docker login --username AWS --password-stdin ${aws_account_id}.dkr.ecr.${aws_region}.amazonaws.com
                        docker tag spring-app:${Docker_tag} ${aws_account_id}.dkr.ecr.${aws_region}.amazonaws.com/spring-app:${Docker_tag}
                        docker push ${aws_account_id}.dkr.ecr.${aws_region}.amazonaws.com/spring-app:${Docker_tag}
                        docker rmi ${aws_account_id}.dkr.ecr.${aws_region}.amazonaws.com/spring-app:${Docker_tag} spring-app:${Docker_tag}
                    '''
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
