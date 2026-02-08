pipeline {
    agent any

    environment {
        PIPELINE_START = "${System.currentTimeMillis()}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo "🔧 Build stage"
                sh '''
                  rm -rf build
                  mkdir -p build
                  cp -r index.html assets build/
                '''
            }
        }

        stage('Test - SonarCloud Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('sonarcloud') {
                        sh """
                          ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=ISI_PROJECT_KEY \
                            -Dsonar.organization=zyynzyy\
                            -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                   DEPLOY_END = System.currentTimeMillis()
                }
                sh '''
                  sudo rm -rf /var/www/html/*
                  sudo cp -r build/* /var/www/html/
                '''
            }
        }
    }

    post {
        success {
            echo "Mantab"

        }

        failure {
            echo "❌ FAILED – not counted in DORA"
        }
    }
}
