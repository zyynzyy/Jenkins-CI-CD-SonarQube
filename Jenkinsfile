pipeline {
    agent any

    tools {
        sonarRunner 'sonar-scanner'
    }

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
                echo "🔧 Building artifact..."
                sh '''
                  rm -rf build
                  mkdir -p build
                  cp -r index.html assets build/
                '''
            }
        }

        stage('Test - SonarCloud Analysis') {
            steps {
                echo "🔍 Running SonarCloud analysis..."
                withSonarQubeEnv('sonarcloud') {
                    sh '''
                      sonar-scanner \
                        -Dsonar.projectKey=ZY_PROJECT_KEY \
                        -Dsonar.organization=ZY_ORG_KEY \
                        -Dsonar.sources=.
                    '''
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
            script {
                def leadTimeMs = DEPLOY_END.toLong() - PIPELINE_START.toLong()
                def leadTimeSec = leadTimeMs / 1000

                echo "📊 DORA METRIC"
                echo "Pipeline Start  : ${PIPELINE_START}"
                echo "Deploy End      : ${DEPLOY_END}"
                echo "Lead Time (sec) : ${leadTimeSec}"
                echo "✅ Deployment SUCCESS (COUNTED)"
            }
        }

        failure {
            echo "❌ Pipeline FAILED – NOT counted in DORA"
        }
    }
}
