pipeline {
    agent any

    environment {
        IMAGE_NAME = 'devops-security-testing'
        IMAGE_TAG = 'v1'
        SONAR_SCANNER = '/opt/sonar-scanner/bin/sonar-scanner'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/subasinik-blip/devops-testing-security.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv venv
                    ./venv/bin/pip install --upgrade pip
                    ./venv/bin/pip install -r requirements.txt
                '''
            }
        }

        stage('Unit Testing - PyTest') {
            steps {
                sh '''
                    ./venv/bin/pytest test_app.py
                '''
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ${SONAR_SCANNER} \
                        -Dsonar.projectKey=devops-security-testing \
                        -Dsonar.projectName=devops-security-testing \
                        -Dsonar.sources=.
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                    cd /tmp
                    trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 1 \
                    ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Run Application') {
            steps {
                sh '''
                    docker rm -f devops-app || true

                    docker run -d \
                    --name devops-app \
                    -p 5000:5000 \
                    ${IMAGE_NAME}:${IMAGE_TAG}

                    sleep 10
                '''
            }
        }

        stage('API Testing - Newman') {
            steps {
                sh '''
                    newman run postman/api-tests.json
                '''
            }
        }

        stage('OWASP ZAP Scan') {
            steps {
                sh '''
                    docker run --rm \
                    --network host \
                    -v "$PWD:/zap/wrk/:rw" \
                    zaproxy/zap-stable \
                    zap-baseline.py \
                    -t http://localhost:5000 \
                    -r zap-report.html || true
                '''
            }
        }

        stage('UI Testing - Selenium') {
            steps {
                sh '''
                    ./venv/bin/pytest selenium/test_ui.py
                '''
            }
        }
    }

    post {
        always {
            sh '''
                docker rm -f devops-app || true
            '''
        }
    }
}
