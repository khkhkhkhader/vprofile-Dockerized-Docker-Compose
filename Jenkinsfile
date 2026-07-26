pipeline {
    agent any

    options {
        // Jenkins لن يعمل Checkout تلقائيًا؛
        // لأننا سننفذه في Stage واضحة.
        skipDefaultCheckout(true)

        // يمنع تشغيل Buildين من نفس الـPipeline معًا.
        disableConcurrentBuilds()
    }

    tools {
        // لازم الاسم يطابق اسم Maven المسجل في:
        // Manage Jenkins → Tools
        maven 'maven for project'
    }

    stages {
        stage('Checkout') {
            steps {
                // يسحب الريبو والـBranch المحددة في إعداد الـJob.
                checkout scm
            }
        }

        stage('Verify Tools') {
            steps {
                sh 'java -version'
                sh 'mvn -version'
            }
        }

        stage('Build and SonarQube Analysis') {
            steps {
                // pom.xml موجود داخل source.
                dir('source') {
                    // يأخذ SonarQube URL والـToken
                    // من إعدادات Jenkins.
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

                    IMAGE_TAG="${BUILD_NUMBER}"

                    docker build \
                      -f docker/database/Dockerfile \
                      -t "vprofile-db:${IMAGE_TAG}" \
                      .

                    docker build \
                      -f docker/rabbitmq/Dockerfile \
                      -t "vprofile-rabbitmq:${IMAGE_TAG}" \
                      .

                    docker build \
                      -f docker/memcached/Dockerfile \
                      -t "vprofile-memcached:${IMAGE_TAG}" \
                      .

                    docker build \
                      -f docker/app/Dockerfile \
                      -t "vprofile-app:${IMAGE_TAG}" \
                      .

                    docker image ls | grep vprofile
                    
                '''
            }
        }
    }

    stage('Trivy Image Scan') {
            steps {
                sh '''
                    set -eux
        
                    IMAGE_TAG="${BUILD_NUMBER}"
                    SCAN_FAILED=0
        
                    for IMAGE in \
                        vprofile-db \
                        vprofile-rabbitmq \
                        vprofile-memcached \
                        vprofile-app
                    do
                        echo "Scanning ${IMAGE}:${IMAGE_TAG}"
        
                        if ! trivy image \
                            --scanners vuln \
                            --ignore-unfixed \
                            --severity HIGH,CRITICAL \
                            --exit-code 1 \
                            "${IMAGE}:${IMAGE_TAG}"
                        then
                            SCAN_FAILED=1
                        fi
                    done
        
                    exit "${SCAN_FAILED}"
                '''
    }
}
    
    post {
        success {
            echo 'Build, SonarQube Quality Gate, and Docker image build passed.'
        }

        failure {
            echo 'Pipeline failed. Check the Console Output.'
        }
    }
}
