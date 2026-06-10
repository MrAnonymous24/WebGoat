pipeline {
    agent any

    environment {
        TARGET_URL        = "http://host.docker.internal:9090/WebGoat"
        ZAP_PORT          = "8090"
        CONTAINER_WEBGOAT = "webgoat"
        CONTAINER_ZAP     = "zap"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Repo cloné — branch: ${env.GIT_BRANCH}"
            }
        }

        stage('SAST - Semgrep') {
            steps {
                sh '''
                    semgrep --config=p/owasp-top-ten \
                            --oss-only \
                            --verbose \
                            . || true
                '''
                sh '''
                    semgrep --config=p/owasp-top-ten \
                            --oss-only \
                            --json \
                            --output semgrep-results.json \
                            . || true
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'semgrep-results.json',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('Build & Run WebGoat') {
            steps {
                sh "docker stop ${CONTAINER_WEBGOAT} || true"
                sh "docker rm   ${CONTAINER_WEBGOAT} || true"
                sh """
                    docker run -d \
                        --name ${CONTAINER_WEBGOAT} \
                        -p 9090:8080 \
                        webgoat/webgoat
                """
                sh 'sleep 30'
                sh 'curl -f http://host.docker.internal:9090/WebGoat || exit 1'
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                sh "docker stop ${CONTAINER_ZAP} || true"
                sh "docker rm   ${CONTAINER_ZAP} || true"
                sh """
                    docker run --rm \
                        --name ${CONTAINER_ZAP} \
                        -u root \
                        -v \${WORKSPACE}:/zap/wrk:rw \
                        ghcr.io/zaproxy/zaproxy:stable \
                        zap-baseline.py \
                            -t ${TARGET_URL} \
                            -r zap-report.html \
                            -I || true
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'zap-report.html',
                                     allowEmptyArchive: true
                }
            }
        }

    }

    post {
        always {
            sh "docker stop ${CONTAINER_WEBGOAT} ${CONTAINER_ZAP} || true"
            sh "docker rm   ${CONTAINER_WEBGOAT} ${CONTAINER_ZAP} || true"

            emailext(
                subject: "Jenkins — Build #${env.BUILD_NUMBER} : ${currentBuild.currentResult}",
                body: """
                    <h2>Rapport DevSecOps — WebGoat</h2>
                    <p><b>Build :</b> #${env.BUILD_NUMBER}</p>
                    <p><b>Statut :</b> ${currentBuild.currentResult}</p>
                    <p><b>Durée :</b> ${currentBuild.durationString}</p>
                    <p><b>Commit :</b> ${env.GIT_COMMIT}</p>
                    <p>Voir les détails : <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                    <hr>
                    <p>Rapports joints : semgrep-results.json + zap-report.html</p>
                """,
                mimeType: 'text/html',
                to: 'babambaye334@gmail.com',
                attachmentsPattern: 'semgrep-results.json, zap-report.html'
            )
        }
        success {
            echo 'Pipeline terminé avec succès.'
        }
        failure {
            echo 'Pipeline en échec.'
        }
    }
}
