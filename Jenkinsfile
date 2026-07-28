pipeline {

    agent {
        label 'devops'
    }

    environment {
        IMAGE_NAME = "docker.io/mahadikbs/spring-petclinic"
        IMAGE_TAG  = "${BUILD_NUMBER}"

        SONARQUBE = "sonarqube"

        DEPLOYMENT = "spring-petclinic"
        NAMESPACE = "spring-petclinic"
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
        //         echo "Building application..."

        //         sh '''
        //         mvn clean compile
        //         '''
        //     }
        // }

        // stage('Unit Test') {
        //     steps {

        //         sh '''
        //         mvn test
        //         '''
        //     }

        //     post {
        //         always {
        //             junit '**/target/surefire-reports/*.xml'
        //         }
        //     }
        // }

        stage('SonarQube Analysis') {

            steps {

                withSonarQubeEnv("${SONARQUBE}") {

                    sh '''
                    mvn sonar:sonar \
                    -Dsonar.projectKey=spring-petclinic \
                    -Dsonar.projectName="Spring PetClinic"
                    '''

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

                sh '''
                mvn package -DskipTests
                '''

            }

        }

        stage('Build Docker Image') {

            steps {

                sh """
                docker build \
                -t ${IMAGE_NAME}:${IMAGE_TAG} \
                -t ${IMAGE_NAME}:latest .
                """

            }

        }

        stage('Docker Login') {

            steps {

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

        stage('Push Docker Image') {

            steps {

                sh """
                docker push ${IMAGE_NAME}:${IMAGE_TAG}
                docker push ${IMAGE_NAME}:latest
                """

            }

        }

        stage('Deploy to Kubernetes') {

            steps {

                sh """
                kubectl set image deployment/${DEPLOYMENT} \
                spring-petclinic=${IMAGE_NAME}:${IMAGE_TAG} \
                -n ${NAMESPACE}
                """

            }

        }

        stage('Verify Deployment') {

            steps {

                sh """
                kubectl rollout status deployment/${DEPLOYMENT} -n ${NAMESPACE}
                kubectl get pods -n ${NAMESPACE}
                """
            }

        }

    }

    post {

        success {

            echo "Deployment Successful"

        }

        failure {

            echo "Pipeline Failed"

        }

        always {

            cleanWs()

        }

    }

}