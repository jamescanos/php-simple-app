pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-cred')
        IMAGE_NAME = "jamescanos/php-simple-app"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/jamescanos/php-simple-app.git'
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    // Obtiene el último commit del repo local
                    def currentCommit = sh(script: "git rev-parse HEAD", returnStdout: true).trim()

                    // Archivo para guardar el último commit desplegado
                    def commitFile = "${env.WORKSPACE}/.last_commit"

                    if (fileExists(commitFile)) {
                        def lastCommit = readFile(commitFile).trim()
                        if (currentCommit == lastCommit) {
                            echo "🟡 No hay cambios nuevos desde el último despliegue (${lastCommit})."
                            currentBuild.result = 'SUCCESS'
                            currentBuild.displayName = "Sin cambios"
                            skipRemainingStages()
                        } else {
                            echo "🟢 Cambios detectados. Último commit anterior: ${lastCommit}"
                        }
                    } else {
                        echo "🚀 Primer despliegue: no existe registro previo de commit."
                    }

                    // Guarda el commit actual para la próxima ejecución
                    writeFile file: commitFile, text: currentCommit
                }
            }
        }

        stage('Generate Tag') {
            when {
                expression { currentBuild.result == null }  // Solo si no se saltó
            }
            steps {
                script {
                    def GIT_COMMIT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    def DATE_TAG = sh(script: "date +%Y%m%d-%H%M%S", returnStdout: true).trim()
                    def VERSION_TAG = "${DATE_TAG}-${GIT_COMMIT}"
                    env.VERSION_TAG = VERSION_TAG
                    echo "🧩 Versión generada: ${VERSION_TAG}"
                }
            }
        }

        stage('Build Docker Image') {
            when {
                expression { currentBuild.result == null }
            }
            steps {
                sh '''
                    echo "=== Construyendo imagen con tag ${VERSION_TAG} ==="
                    docker build -t $IMAGE_NAME:$VERSION_TAG .
                '''
            }
        }

        stage('Push to DockerHub') {
            when {
                expression { currentBuild.result == null }
            }
            steps {
                sh '''
                    echo "=== Iniciando sesión en DockerHub ==="
                    echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin

                    echo "=== Subiendo imagen con tag ${VERSION_TAG} ==="
                    docker push $IMAGE_NAME:$VERSION_TAG
                '''
            }
        }
    }

    post {
        always {
            echo "🧹 Limpieza final"
            sh 'docker system prune -f || true'
        }
        success {
            echo "✅ Pipeline completado con éxito"
        }
        failure {
            echo "❌ Pipeline falló"
        }
    }
}
