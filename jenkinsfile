pipeline {
  agent any

  environment {
    PIPELINE_START = System.currentTimeMillis()
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build') {
      steps {
        sh '''
          mkdir -p build
          cp -r index.html assets build/
        '''
      }
    }

    stage('Test - SonarCloud Analysis') {
      steps {
        withSonarQubeEnv('sonarcloud') {
          sh '''
            sonar-scanner \
              -Dsonar.login=$SONAR_TOKEN
          '''
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          script {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
              error "Quality Gate FAILED: ${qg.status}"
            }
          }
        }
      }
    }

    stage('Deploy') {
      environment {
        DEPLOY_START = System.currentTimeMillis()
      }
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
      script {
        DEPLOY_END = System.currentTimeMillis()
        def leadTime = DEPLOY_END - PIPELINE_START
        echo "📊 Lead Time for Change (ms): ${leadTime}"
        echo "✅ Deployment SUCCESS"
      }
    }

    failure {
      echo "❌ Pipeline FAILED – not counted in DORA"
    }
  }
}
