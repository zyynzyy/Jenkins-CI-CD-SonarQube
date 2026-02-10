pipeline {
    agent any

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
                          -Dsonar.projectKey=zyynzyy_Jenkins-CI-CD-SonarQube \
                          -Dsonar.organization=zyynzyy \
                          -Dsonar.sources=.
                        """
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

        stage('Deploy') {
            steps {
                sh '''
                  sudo rm -rf /var/www/html/*
                  sudo cp -r build/* /var/www/html/
                '''
            }
        }
    }

    post {
        success {
            echo "✅ PIPELINE SUCCESS"
        }

        failure {
            echo "❌ PIPELINE FAILED"
        }
    }
}
