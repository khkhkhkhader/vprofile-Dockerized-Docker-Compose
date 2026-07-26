// make sure to install aws cli v2 , trivy , docker , add jenkins user to deocker group and activate user membership , sonarqube
// installed pluigins SonarQube Scanner , Credentials Binding,  GitHub Integration , Docker Pipeline 
// maven in jenkins tools 3.9.16
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
    }

    tools {
        maven 'maven for project'
    }

    environment {
        AWS_REGION   = 'us-east-2'
        ECR_REGISTRY = '578620461945.dkr.ecr.us-east-2.amazonaws.com'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Set Image Tag') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Full Git Commit SHA: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Verify Tools') {
            steps {
                sh 'java -version'
                sh 'mvn -version'
                sh 'docker version'
                sh 'trivy --version'
                sh 'aws --version'
            }
        }

        stage('Build and SonarQube Analysis') {
            steps {
                dir('source') {
                    withSonarQubeEnv('sonarqube') {
                        sh '''
                            mvn -B -ntp clean verify \
                              org.sonarsource.scanner.maven:sonar-maven-plugin:5.5.0.6356:sonar \
                              -Dsonar.projectKey=production \
                              -Dsonar.projectName=production
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -eux

                    docker build \
                      -f docker/database/Dockerfile \
                      -t "${ECR_REGISTRY}/vprofile/db:${IMAGE_TAG}" \
                      .

                    docker build \
                      -f docker/rabbitmq/Dockerfile \
                      -t "${ECR_REGISTRY}/vprofile/rabbitmq:${IMAGE_TAG}" \
                      .

                    docker build \
                      -f docker/memcached/Dockerfile \
                      -t "${ECR_REGISTRY}/vprofile/memcached:${IMAGE_TAG}" \
                      .

                    docker build \
                      -f docker/app/Dockerfile \
                      -t "${ECR_REGISTRY}/vprofile/app:${IMAGE_TAG}" \
                      .

                    echo "Built Docker images:"
                    docker image ls | grep vprofile
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    set -eux

                    for REPOSITORY in \
                        vprofile/db \
                        vprofile/rabbitmq \
                        vprofile/memcached \
                        vprofile/app
                    do
                        IMAGE="${ECR_REGISTRY}/${REPOSITORY}:${IMAGE_TAG}"

                        echo "Scanning ${IMAGE}"

                        trivy image \
                          --scanners vuln \
                          --severity HIGH,CRITICAL \
                          --exit-code 0 \
                          "${IMAGE}"
                    done
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                    set -eu

                    echo "Logging in to ECR: ${ECR_REGISTRY}"

                    aws ecr get-login-password \
                      --region "${AWS_REGION}" \
                      | docker login \
                          --username AWS \
                          --password-stdin "${ECR_REGISTRY}"

                    for REPOSITORY in \
                        vprofile/app \
                        vprofile/db \
                        vprofile/memcached \
                        vprofile/rabbitmq
                    do
                        IMAGE="${ECR_REGISTRY}/${REPOSITORY}:${IMAGE_TAG}"

                        echo "Pushing ${IMAGE}"

                        docker push "${IMAGE}"
                    done
                '''
            }
        }

        stage('Cleanup Local Images') {
            steps {
                sh '''
                    set -u

                    for REPOSITORY in \
                        vprofile/app \
                        vprofile/db \
                        vprofile/memcached \
                        vprofile/rabbitmq
                    do
                        IMAGE="${ECR_REGISTRY}/${REPOSITORY}:${IMAGE_TAG}"

                        echo "Removing local image: ${IMAGE}"

                        docker image rm "${IMAGE}" || true
                    done

                    echo "Removing unused dangling image layers"

                    docker image prune -f || true

                    echo "Local Docker image cleanup completed"
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
            echo "Images pushed using commit tag: ${env.IMAGE_TAG}"
        }

        failure {
            echo 'Pipeline failed. Check the Console Output.'
        }

        always {
            echo "Pipeline finished for Build Number: ${BUILD_NUMBER}"
        }
    }
}
