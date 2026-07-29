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

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out source..."
                checkout scm
            }
        }

        // stage('Build') {
        //     steps {
        //         container('maven') {
        //             sh '''
        //                 mvn clean compile
        //             '''
        //         }
        //     }
        // }

        // stage('Unit Test') {
        //     steps {
        //         container('maven') {
        //             sh '''
        //                 mvn test
        //             '''
        //         }
        //     }
        //     post {
        //         always {
        //             junit '**/target/surefire-reports/*.xml'
        //         }
        //     }
        // }

        // stage('SonarQube Analysis') {
        //     steps {
        //         container('maven') {
        //             withSonarQubeEnv('sonarqube') {
        //                 sh '''
        //                       mvn clean verify \
        //                       org.sonarsource.scanner.maven:sonar-maven-plugin:5.2.0.4988:sonar \
        //                       -Dsonar.projectKey=spring-petclinic \
        //                       -Dsonar.projectName="Spring PetClinic"
        //                 '''
        //             }
        //         }
        //     }
        // }   
        // stage('Quality Gate') {
        //     steps {
        //         timeout(time: 15, unit: 'MINUTES') {
        //             waitForQualityGate abortPipeline: true
        //         }
        //     }
        // }

        // stage('Package') {
        //     steps {
        //         container('maven') {
        //             sh '''
        //                 mvn package -DskipTests
        //             '''
        //         }
        //     }
        // }

        stage('Docker Login/Build/push') {
            steps {
                    container('kaniko') {
                        withCredentials([
                            usernamePassword(
                                credentialsId: 'dockerhub',
                                usernameVariable: 'DOCKER_USER',
                                passwordVariable: 'DOCKER_PASS'
                            )
                        ]) {
                            sh '''
                                mkdir -p /tmp/.docker

                                AUTH=$(printf "%s:%s" "$DOCKER_USER" "$DOCKER_PASS" | base64 | tr -d '\n')

                                cat > /tmp/.docker/config.json <<EOF
                    {
                    "auths": {
                        "https://index.docker.io/v1/": {
                        "auth": "$AUTH"
                        }
                    }
                    }
                    EOF

                                export DOCKER_CONFIG=/tmp/.docker

                                /kaniko/executor \
                                --context="${WORKSPACE}" \
                                --dockerfile="${WORKSPACE}/Dockerfile" \
                                --destination=docker.io/mahadikbs/spring-petclinic:${BUILD_NUMBER} \
                                --destination=docker.io/mahadikbs/spring-petclinic:latest \
                                --cache=true
                            '''
                        }
                    }
            }
        }    

        // stage('Build Docker Image') {
        //     steps {
        //         container('docker') {
        //             sh """
        //                 docker build \
        //                   -t ${IMAGE_NAME}:${IMAGE_TAG} \
        //                   -t ${IMAGE_NAME}:latest .
        //             """
        //         }
        //     }
        // }

        // stage('Push Docker Image') {
        //     steps {
        //         container('docker') {
        //             sh """
        //                 docker push ${IMAGE_NAME}:${IMAGE_TAG}
        //                 docker push ${IMAGE_NAME}:latest
        //             """
        //         }
        //     }
        // }

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