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
        SONAR_TOKEN = credentials('sonar-token')
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

                    echo "Commit: ${env.GIT_COMMIT_SHORT}"
                    echo "Commit time: ${env.GIT_COMMIT_ISO}"
                }
            }
        }

        stage('Build') {
            steps {
                echo "Build static web"
                sh '''
                    rm -rf build
                    mkdir -p build
                    cp -r index.html assets build/
                '''
            }
        }

        stage('SonarCloud Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'

                    withSonarQubeEnv('sonarcloud') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                              -Dsonar.organization=${SONAR_ORG} \
                              -Dsonar.sources=. \
                              -Dsonar.host.url=https://sonarcloud.io \
                              -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('Quality Gate (Manual - SonarCloud API)') {
            steps {
                script {
                    echo "Menunggu hasil analisis SonarCloud..."
                    sleep 10

                    def status = sh(
                        script: """
                            curl -s -u ${SONAR_TOKEN}: \
                            "https://sonarcloud.io/api/qualitygates/project_status?projectKey=${SONAR_PROJECT_KEY}" \
                            | grep -o '"status":"[^"]*"' | cut -d':' -f2 | tr -d '"'
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Quality Gate Status: ${status}"

                    if (status != "OK") {
                        error "Quality Gate FAILED!"
                    }
                }
            }
        }

        stage('Deploy to Nginx') {
            steps {
                echo "Deploy ke Nginx"
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

                    env.DORA_LT_SECONDS = ltSeconds.toString()
                    env.DORA_LT_MINUTES = String.format('%.2f', ltMinutes)
                    env.DORA_DF_COUNT = deployCount.toString()
                    env.DORA_DF_PER_DAY = String.format('%.4f', dfPerDay)

                    writeFile file: 'dora-metrics.json', text: groovy.json.JsonOutput.prettyPrint(
                        groovy.json.JsonOutput.toJson([
                            buildNumber: env.BUILD_NUMBER,
                            commit: env.GIT_COMMIT_SHORT,
                            leadTimeSeconds: ltSeconds,
                            leadTimeMinutes: ltMinutes,
                            deployCountLast30Days: deployCount,
                            deployFrequencyPerDay: dfPerDay
                        ])
                    )

                    archiveArtifacts artifacts: 'dora-metrics.json', fingerprint: true

                    currentBuild.description = "LT=${env.DORA_LT_MINUTES} min | DF30=${env.DORA_DF_COUNT}"
                }
            }
        }
    }

    post {
        success {
            echo "=============================="
            echo "PIPELINE SUCCESS"
            echo "=============================="

            echo "DORA FINAL RESULT:"
            echo "Lead Time (LT): ${env.DORA_LT_SECONDS} detik (${env.DORA_LT_MINUTES} menit)"
            echo "Deployment Frequency (DF): ${env.DORA_DF_COUNT} deploy dalam ${env.DORA_WINDOW_DAYS} hari"
            echo "DF Rate: ${env.DORA_DF_PER_DAY} deploy/hari"
        }

        failure {
            echo "PIPELINE FAILED"
            echo "Kemungkinan penyebab:"
            echo "- Quality Gate gagal"
            echo "- SonarCloud belum selesai analisis"
        }

        always {
            echo "Build status: ${currentBuild.currentResult}"
        }
    }
}
