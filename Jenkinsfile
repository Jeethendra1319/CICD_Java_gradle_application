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
                        withCredentials([usernamePassword(credentialsId: 'aws-login-cred', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
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

        stage('Prepare Helm Charts') {
            steps {
                script {
                    docker.image('alpine:3.18').inside('--user root') {
                        withCredentials([usernamePassword(credentialsId: 'aws-login-cred', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                            sh '''
                                apk add --no-cache bash curl git tar gzip python3 py3-pip
                                pip install --no-cache-dir awscli
                                curl -fsSL https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz -o helm.tar.gz
                                tar -xzvf helm.tar.gz -C /tmp
                                mv /tmp/linux-amd64/helm /usr/local/bin/helm

                                sed -i "s:IMAGE_NAME:${aws_account_id}.dkr.ecr.${aws_region}.amazonaws.com/spring-app:" kubernetes/myapp/values.yaml
                                sed -i "s:IMAGE_TAG:${Docker_tag}:" kubernetes/myapp/values.yaml
                                helm package kubernetes/myapp/

                                helmversion=$(helm show chart kubernetes/myapp/ | grep version | cut -d: -f 2 | tr -d ' ')
                                aws s3 cp myapp-${helmversion}.tgz s3://helm-chart-testing/helm-charts/spring-app-${helmversion}.tgz
                            '''
                        }
                    }
                }
            }
        }

        stage('Deploy to EKS Cluster') {
            steps {
                script {
                    docker.image('707077521494.dkr.ecr.us-east-1.amazonaws.com/spring-app:deploy').inside('--entrypoint="" --user root') {
                        withCredentials([usernamePassword(credentialsId: 'aws-login-cred', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                            sh '''
                                mkdir -p /root/.aws
                                echo "[default]" > /root/.aws/config
                                echo "region = us-east-1" >> /root/.aws/config
                                export AWS_CONFIG_FILE="/root/.aws/config"
                                aws eks update-kubeconfig --region ${aws_region} --name my-k8s-cluster
                                helm upgrade --install myjavaapp kubernetes/myapp/
                                helm list
                                sleep 120
                                kubectl get po
                            '''
                        }
                    }
                }
            }
        }

        stage('Verify App Deployment') {
            steps {
                script {
                    docker.image('707077521494.dkr.ecr.us-east-1.amazonaws.com/spring-app:deploy').inside('--entrypoint="" --user root') {
                        withCredentials([usernamePassword(credentialsId: 'aws-login-cred', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                            sh '''
                                mkdir -p /root/.aws
                                echo "[default]" > /root/.aws/config
                                echo "region = us-east-1" >> /root/.aws/config
                                export AWS_CONFIG_FILE="/root/.aws/config"
                                aws eks update-kubeconfig --region ${aws_region} --name my-k8s-cluster
                                kubectl run curl --image=curlimages/curl -i --rm --restart=Never -- curl myjavaapp-spring-app:8080
                            '''
                        }
                    }
                }
            }

            post {
                always {
                    script {
                        docker.image('707077521494.dkr.ecr.us-east-1.amazonaws.com/spring-app:deploy').inside('--entrypoint="" --user root') {
                            withCredentials([usernamePassword(credentialsId: 'aws-login-cred', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                                sh '''
                                    mkdir -p /root/.aws
                                    echo "[default]" > /root/.aws/config
                                    echo "region = us-east-1" >> /root/.aws/config
                                    export AWS_CONFIG_FILE="/root/.aws/config"
                                    aws eks update-kubeconfig --region ${aws_region} --name my-k8s-cluster
                                    helm uninstall myjavaapp
                                '''
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'build/reports/tests/test/**', followSymlinks: false
            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'build/reports/tests/test/', reportFiles: 'index.html', reportName: 'test-case-report', reportTitles: 'test-case-report', useWrapperFileDirectly: true])
            cleanWs()
        }

        success {
            echo "Build succeeded!"
        }

        failure {
            echo "Build failed, investigate the errors above."
        }

        unstable {
            echo "Build is unstable, please review the warnings and issues."
        }
    }
}
