pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = "aelkowidy"
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/service-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
        FULL_IMAGE = "${IMAGE_NAME}:${IMAGE_TAG}"
        REPO_URL = "https://github.com/aelkowidy/digi-jenkins-lab.git"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[
                        url: "${REPO_URL}",
                        credentialsId: 'github-pat-creds'
                    ]]
                ])
            }
        }

        stage('Run Unit Tests') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '/usr/local/bin/phpunit --log-junit results.xml tests/'
                }
            }
        }

        stage('Display Results') {
            steps {
                junit 'results.xml'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=service-app \
                        -Dsonar.projectName=service-app \
                        -Dsonar.sources=src \
                        -Dsonar.php.tests.reportPath=results.xml
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${FULL_IMAGE} .
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    trivy image --exit-code 1 --severity CRITICAL ${FULL_IMAGE}
                '''
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_PASS'
                )]) {
                    sh '''
                        echo "$DOCKERHUB_PASS" | docker login -u "$DOCKERHUB_USER" --password-stdin
                        docker push ${FULL_IMAGE}
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker rm -f service-app || true
                    docker run -d --name service-app -p 8081:80 ${FULL_IMAGE} php -S 0.0.0.0:80 index.php
                '''
            }
        }
    }

    post {
        always {
            sh '''
                docker rmi ${FULL_IMAGE} || true
                docker image prune -f || true
            '''
            echo 'Pipeline job finished.'
        }

        success {
            echo 'Congratulations! Full Lab 2 pipeline passed successfully.'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins console output for details.'
        }
    }
}                catchError(buildResu
