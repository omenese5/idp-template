pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'URL del repositorio Cliente')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Rama a desplegar')
        string(name: 'LANGUAGE', defaultValue: 'node', description: 'Lenguaje (node/java/angular)')
        string(name: 'EMAIL_TO', defaultValue: '', description: 'Email de notificación')
        string(name: 'HOST_PORT', defaultValue: '8085', description: 'Puerto externo del servidor')
    }

    environment {
        TEMPLATES_REPO = "https://github.com/omenese5/idp-template.git" 
    }

    stages {
        stage('1. Preparar Workspace') {
            steps {
                script {
                    def urlParts = params.REPO_URL.tokenize('/')
                    def repoName = urlParts.last().replace('.git', '').toLowerCase()
                    env.APP_NAME = repoName
                    
                    echo "--- Resumen del Despliegue ---"
                    echo "App: ${env.APP_NAME}"
                    
                    cleanWs()
                    dir('app') { git branch: params.BRANCH, url: params.REPO_URL }
                    dir('tooling') { git branch: 'main', url: env.TEMPLATES_REPO }
                }
            }
        }

        stage('2. Construcción') {
            steps {
                script {
                    // Stage 2: Build
                    sh "cp tooling/docker/${params.LANGUAGE}/Dockerfile app/Dockerfile"
                    dir('app') {
                        sh "docker build --network host -t ${env.APP_NAME}:${env.BUILD_NUMBER} ."
                    }
                }
            }
        }

        stage('3. Despliegue') {
            steps {
                script {
                    // Stage 3: Dockerize/Deploy
                    echo "Desplegando aplicación ${env.APP_NAME} en puerto ${params.HOST_PORT}..."
                    
                    withEnv([
                        "APP_IMAGE=${env.APP_NAME}:${env.BUILD_NUMBER}",
                        "APP_PORT=${params.HOST_PORT}", 
                        "ENVIRONMENT=${params.BRANCH}",
                        "DB_HOST=postgres_db", 
                        "DB_NAME=mirai_db",
                        "DB_USER=postgres",
                        "DB_PASS=donlito123"
                    ]) {
                        dir('tooling/templates') {
                            sh "curl -SL https://github.com/docker/compose/releases/download/v2.29.1/docker-compose-linux-x86_64 -o docker-compose"
                            sh "chmod +x docker-compose"

                            sh """
                                CONFLICT_ID=\$(docker ps -q --filter publish=${params.HOST_PORT})
                                if [ ! -z "\$CONFLICT_ID" ]; then
                                    docker rm -f \$CONFLICT_ID
                                fi
                            """

                            sh "./docker-compose -p ${env.APP_NAME} -f docker-compose.yml down || true"
                            sh "./docker-compose -p ${env.APP_NAME} -f docker-compose.yml up -d"
                            
                            // Un pequeño sleep para asegurar que el contenedor arrancó antes de notificar
                            sh "sleep 5"
                        }
                    }
                }
            }
        }

        // --- AQUÍ ESTÁ EL CAMBIO: El email ahora es un Stage oficial ---
        stage('4. Notificación') {
            steps {
                script {
                    echo "--- Enviando Notificación de Éxito ---"
                    mail(
                        to: "${params.EMAIL_TO}",
                        subject: "Despliegue Exitoso: ${env.APP_NAME} v${env.BUILD_NUMBER}",
                        mimeType: 'text/html',
                        body: """
                            <div style="font-family: Arial, sans-serif; padding: 20px; border: 1px solid #e0e0e0; border-radius: 5px;">
                                <h2 style="color: #2da44e;">Zalgodyne IDP - Despliegue Completado</h2>
                                <p>El proyecto <b>${env.APP_NAME}</b> está online.</p>
                                <ul>
                                    <li><b>Build ID:</b> #${env.BUILD_NUMBER}</li>
                                </ul>
                            </div>
                        """
                    )
                }
            }
        }
    }

    // Mantenemos 'post failure' para avisar SI ALGO FALLA antes de llegar al stage 4
    post {
        failure {
            script {
                echo "Pipeline Fallido. Enviando alerta..."
                mail(
                    to: "${params.EMAIL_TO}",
                    subject: "Falló el Despliegue: ${env.APP_NAME}",
                    mimeType: 'text/html',
                    body: "<p>Hubo un error en el despliegue. Revisa los logs en Jenkins.</p>"
                )
            }
        }
    }
}