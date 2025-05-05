pipeline {
    agent any

    environment {
        PROJECT_NAME = 'jenkins-example'
        REPORT_DIR = 'build/test-results/test'
        JACOCO_HTML = 'build/reports/jacoco/test/html'
        EMAIL_RECIPIENTS = 'team@example.com'
        TELEGRAM_CHAT_ID = credentials('TELEGRAM_CHAT_ID')
        TELEGRAM_TOKEN = credentials('TELEGRAM_TOKEN')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        timestamps()
    }

    stages {
        stage('Инициализация') {
            steps {
                echo "Запуск пайплайна для проекта: ${PROJECT_NAME}"
            }
        }

        stage('Получение исходников') {
            steps {
                checkout scm
            }
        }

        stage('Сборка проекта') {
            steps {
                sh './gradlew clean build -x test'
            }
        }

        stage('Запуск тестов') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh './gradlew test'
                }
            }
            post {
                always {
                    junit "${REPORT_DIR}/*.xml"
                }
            }
        }

        stage('Генерация метрик') {
            steps {
                echo "Ключевые метрики сборки:"
                echo "- Длительность: ${currentBuild.durationString}"
                echo "- Автор: ${env.BUILD_USER ?: 'N/A'}"
                echo "- Статус: ${currentBuild.currentResult}"
                echo "- Коммит: ${env.GIT_COMMIT ?: 'N/A'}"
            }
        }

        stage('Отчёт в HTML') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    publishHTML(target: [
                        reportDir: "${JACOCO_HTML}",
                        reportFiles: 'index.html',
                        reportName: 'Jacoco Code Coverage',
                        keepAll: true,
                        alwaysLinkToLastBuild: true,
                        allowMissing: true
                    ])
                }
            }
        }

        stage('Уведомление по Email') {
            steps {
                script {
                    try {
                        emailext(
                            subject: "Сборка ${env.JOB_NAME} #${env.BUILD_NUMBER}: ${currentBuild.currentResult}",
                            body: """<p>Статус: ${currentBuild.currentResult}</p>
                                     <p>Ссылка на билд: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>""",
                            recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                            to: "${EMAIL_RECIPIENTS}",
                            mimeType: 'text/html'
                        )
                    } catch (err) {
                        echo "Ошибка отправки email: ${err.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }

        stage('Уведомление в Telegram') {
            steps {
                script {
                    try {
                        sh(
                            label: 'Отправка уведомления в Telegram',
                            script: """
                                curl -s -X POST https://api.telegram.org/bot${TELEGRAM_TOKEN}/sendMessage \\
                                    --data-urlencode chat_id=${TELEGRAM_CHAT_ID} \\
                                    --data-urlencode text="*Сборка:* ${JOB_NAME} #${BUILD_NUMBER}\\n*Статус:* ${BUILD_STATUS:-$BUILD_RESULT}\\n*Ссылка:* ${BUILD_URL}" \\
                                    -d parse_mode=Markdown
                            """
                        )
                    } catch (err) {
                        echo "Ошибка отправки в Telegram: ${err.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Сборка завершена успешно."
        }
        failure {
            echo "Сборка завершилась с ошибкой."
        }
    }
}
