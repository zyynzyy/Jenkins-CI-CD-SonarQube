pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        APP_DIR = '/var/www/dora-site'
        DORA_LOG = '/var/lib/jenkins/dora-metrics/deployments.csv'
        DORA_WINDOW_DAYS = '30'
        SONAR_PROJECT_KEY = 'zyynzyy_Jenkins-CI-CD-SonarQube'
        SONAR_ORG = 'zyynzyy'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Capture Source Info') {
            steps {
                script {
                    env.GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.GIT_COMMIT_EPOCH = sh(script: "git log -1 --format=%ct", returnStdout: true).trim()
                    env.GIT_COMMIT_ISO = sh(script: "git log -1 --format=%cI", returnStdout: true).trim()
                }
            }
        }

        stage('Build') {
            steps {
                sh '''
                    rm -rf build
                    mkdir -p build
                    cp -r index.html assets build/
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('sonarcloud') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.organization=${SONAR_ORG} \
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

        stage('Deploy to Nginx') {
            steps {
                sh '''
                    mkdir -p "$APP_DIR"
                    rm -rf "$APP_DIR"/*
                    cp -r build/* "$APP_DIR"/
                '''
            }
        }

        stage('DORA Metrics') {
            steps {
                script {
                    def deployEndEpoch = sh(script: "date +%s", returnStdout: true).trim().toInteger()
                    def commitEpoch = env.GIT_COMMIT_EPOCH.toInteger()
                    def ltSeconds = deployEndEpoch - commitEpoch
                    def ltMinutes = ltSeconds / 60.0

                    sh """
                        mkdir -p \$(dirname "$DORA_LOG")
                        if [ ! -f "$DORA_LOG" ]; then
                          echo "build_number,commit,commit_epoch,deploy_epoch,lt_seconds,status" > "$DORA_LOG"
                        fi
                        printf '%s,%s,%s,%s,%s,%s\\n' \
                          '${env.BUILD_NUMBER}' \
                          '${env.GIT_COMMIT_SHORT}' \
                          '${env.GIT_COMMIT_EPOCH}' \
                          '${deployEndEpoch}' \
                          '${ltSeconds}' \
                          'SUCCESS' >> "$DORA_LOG"
                    """

                    def windowStart = deployEndEpoch - (env.DORA_WINDOW_DAYS.toInteger() * 24 * 3600)

                    def deployCount = sh(
                        script: """
                            awk -F',' -v ws=${windowStart} '
                                NR > 1 && \$4 >= ws && \$6 == "SUCCESS" { c++ }
                                END { print c+0 }
                            ' "$DORA_LOG"
                        """,
                        returnStdout: true
                    ).trim().toInteger()

                    def dfPerDay = deployCount / env.DORA_WINDOW_DAYS.toInteger().toDouble()

                    writeFile file: 'dora-metrics.json', text: groovy.json.JsonOutput.prettyPrint(
                        groovy.json.JsonOutput.toJson([
                            buildNumber: env.BUILD_NUMBER,
                            commit: env.GIT_COMMIT_SHORT,
                            commitEpoch: commitEpoch,
                            deployEpoch: deployEndEpoch,
                            leadTimeSeconds: ltSeconds,
                            leadTimeMinutes: ltMinutes,
                            deployCountLast30Days: deployCount,
                            deployFrequencyPerDay: dfPerDay
                        ])
                    )

                    archiveArtifacts artifacts: 'dora-metrics.json', fingerprint: true
                    currentBuild.description = "LT=${String.format('%.2f', ltMinutes)} min | DF30=${deployCount}"

                    echo "LT (Lead Time): ${ltSeconds} detik"
                    echo "DF (Deployment Frequency): ${deployCount} deployment sukses dalam ${env.DORA_WINDOW_DAYS} hari"
                }
            }
        }
    }

    post {
        success {
            echo "PIPELINE SUCCESS"
        }
        failure {
            echo "PIPELINE FAILED"
        }
        always {
            echo "Build finished with status: ${currentBuild.currentResult}"
        }
    }
}
