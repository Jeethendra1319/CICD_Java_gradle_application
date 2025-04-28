def getDockerTag() {
    return sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
}

def getAwsAccountID() {
    return sh(script: 'aws sts get-caller-identity --query Account --output text', returnStdout: true).trim()
}

pipeline {
    agent {
        label 'ec2-fleet'
    }

    environment {
        Docker_tag = getDockerTag()
        aws_account_id = getAwsAccountID()
        aws_region = "us-east-1"
    }

    stages {

        stage('Initial Checks') {
            parallel {
                stage('Lint') {
                    steps {
                        script {
                            docker.image('438465167406.dkr.ecr.us-east-1.amazonaws.com/spring-app:lint').inside('--user root') {
                                try {
                                    sh 'chmod +x lint-all.sh'
                                    sh './lint-all.sh'
                                } catch (err) {
                                    currentBuild.result = 'UNSTABLE'
                                    echo "Please correct linter issues."
                                    return
                                }
                            }
                        }
                    }
                }

                stage('Health Check') {
                    steps {
                        script {
                            sh 'chmod +x health-check.sh'
                            sh './health-check.sh'
                        }
                    }
                }
            }
        }

        stage('Build and Sonar Parallel') {
            parallel {
                stage('Build') {
                    steps {
                        script {
                            docker.image('openjdk:11').inside('--user root') {
                                sh 'chmod +x gradlew'
                                sh 'whoami'
                                sh './gradlew build'
                            }
                        }
                    }
                }

                stage('Sonar Scan') {
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
                                    echo "SonarQube scan failed, marking build as UNSTABLE. Error: ${err}"
                                    return
                                }

                                timeout(time: 1, unit: 'HOURS') {
                                    def qg = waitForQualityGate()
                                    if (qg.status != 'OK') {
                                        error "Pipeline aborted due to quality gate failure: ${qg.status}"
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
                    sh 'docker build -t spring-app:${Docker_tag} .'
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

        stage('Prepare Helm Charts') {
            steps {
                script {
                    sh """
                        sed -i "s:IMAGE_NAME:\${aws_account_id}.dkr.ecr.\${aws_region}.amazonaws.com/spring-app:" kubernetes/myapp/values.yaml
                        sed -i "s:IMAGE_TAG:\${Docker_tag}:" kubernetes/myapp/values.yaml
                        helm package kubernetes/myapp/
                        helmversion=\$( helm show chart kubernetes/myapp/ | grep version | cut -d: -f 2 | tr -d ' ' )
                        aws s3 cp spring-app-\${helmversion}.tgz s3://nimbus-python-practice/helm-charts/spring-app-\${helmversion}.tgz
                    """
                }
            }
        }

        stage('Deploy to EKS Cluster') {
            steps {
                script {
                    dir('kubernetes') {
                        docker.image('438465167406.dkr.ecr.us-east-1.amazonaws.com/spring-app:deploy').inside('--user root') {
                            withCredentials([usernamePassword(credentialsId: 'aws-login-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                                sh '''
                                    mkdir -p /root/.aws
                                    echo "[default]" > /root/.aws/config
                                    echo "region = us-east-1" >> /root/.aws/config
                                    export AWS_CONFIG_FILE="/root/.aws/config"
                                    aws eks update-kubeconfig --region ${aws_region} --name my-k8s-cluster
                                    helm upgrade --install myjavaapp myapp/
                                    helm list
                                    sleep 120
                                    kubectl get po
                                '''
                            }
                        }
                    }
                }
            }
        }

        stage('Verify App Deployment') {
            steps {
                script {
                    docker.image('438465167406.dkr.ecr.us-east-1.amazonaws.com/spring-app:deploy').inside('--user root') {
                        withCredentials([usernamePassword(credentialsId: 'aws-login-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
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
            docker.image('438465167406.dkr.ecr.us-east-1.amazonaws.com/spring-app:deploy').inside('--user root') {
                withCredentials([usernamePassword(credentialsId: 'aws-login-creds', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
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
            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'build/reports/tests/test/', reportFiles: 'index.html', reportName: 'Test Case Report', reportTitles: 'Test Case Report', useWrapperFileDirectly: true])
            cleanWs()
        }
    }
}
