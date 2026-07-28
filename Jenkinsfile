pipeline {

    agent {
        label 'devops'
    }

    environment {
        IMAGE_NAME = "docker.io/mahadikbs/spring-petclinic"
        IMAGE_TAG  = "${BUILD_NUMBER}"

        SONARQUBE = "sonarqube"

        DEPLOYMENT = "spring-petclinic"
        NAMESPACE  = "spring-petclinic"
    }

    options {
        timestamps()
        ansiColor('xterm')
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out source..."
                checkout scm
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    sh '''
                        mvn clean compile
                    '''
                }
            }
        }

        stage('Unit Test') {
            steps {
                container('maven') {
                    sh '''
                        mvn test
                    '''
                }
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('maven') {
                    withSonarQubeEnv('sonarqube') {
                        sh '''
                            mvn sonar:sonar \
                              -Dsonar.projectKey=spring-petclinic \
                              -Dsonar.projectName="Spring PetClinic"
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package') {
            steps {
                container('maven') {
                    sh '''
                        mvn package -DskipTests
                    '''
                }
            }
        }

        stage('Docker Login') {
            steps {
                container('docker') {
                    withCredentials([
                        usernamePassword(
                            credentialsId: 'dockerhub',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh '''
                            echo "$DOCKER_PASS" | docker login \
                              -u "$DOCKER_USER" \
                              --password-stdin
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('docker') {
                    sh """
                        docker build \
                          -t ${IMAGE_NAME}:${IMAGE_TAG} \
                          -t ${IMAGE_NAME}:latest .
                    """
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                container('docker') {
                    sh """
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    sh """
                        kubectl set image deployment/${DEPLOYMENT} \
                        spring-petclinic=${IMAGE_NAME}:${IMAGE_TAG} \
                        -n ${NAMESPACE}

                        kubectl rollout status deployment/${DEPLOYMENT} \
                        -n ${NAMESPACE}
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                container('kubectl') {
                    sh '''
                        kubectl get pods -n spring-petclinic
                        kubectl get svc -n spring-petclinic
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
        }

        failure {
            echo "Pipeline failed."
        }

        always {
            cleanWs()
        }
    }
}