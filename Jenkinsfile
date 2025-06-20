pipeline {
    agent any

    environment {
        PROJECT_NAME = 'jenkins-example'
        REPORT_DIR = 'build/test-results/test'
        JACOCO_HTML = 'build/reports/jacoco/test/html'
        EMAIL_RECIPIENTS = 'zhi13@yandex.ru' // нужно заменить на валидный email
        EMAIL_FROM = 'zhi13@yandex.ru' // нужно заменить на валидный email
        EMAIL_SUBJECT = 'Результат сборки Jenkins'
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
                sh '.\gradlew clean build -x test'
            }
        }

        stage('Запуск тестов') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh '.\gradlew test'
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

        stage('Уведомления') {
            steps {
                script {
                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        emailext(
                            subject: "${EMAIL_SUBJECT}",
                            body: """<p>Статус: ${currentBuild.currentResult}</p>
                                     <p>Ссылка на билд: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>""",
                            recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                            to: "${EMAIL_RECIPIENTS}",
                            from: "${EMAIL_FROM}",
                            mimeType: 'text/html'
                        )
                    }

                    catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                        withEnv(["CHAT_ID=${TELEGRAM_CHAT_ID}", "TOKEN=${TELEGRAM_TOKEN}"]) {
                            sh '''
                                curl -s -X POST https://api.telegram.org/bot$TOKEN/sendMessage \
                                    --data-urlencode chat_id=$CHAT_ID \
                                    --data-urlencode text="Сборка: ${JOB_NAME}/${BRANCH_NAME} #${BUILD_NUMBER}\nСтатус: ${BUILD_STATUS}\nСсылка: ${BUILD_URL}" \
                                    -d parse_mode=HTML
                            '''
                        }
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