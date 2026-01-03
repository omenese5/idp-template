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
        // Asegúrate de que este repo tenga las carpetas docker/node y docker/angular
        TEMPLATES_REPO = "https://github.com/omenese5/idp-template.git" 
    }

    stages {
        stage('1. Preparar Workspace') {
            steps {
                script {
                    // Extraer nombre del repo para usarlo como ID de la app
                    def urlParts = params.REPO_URL.tokenize('/')
                    def repoName = urlParts.last().replace('.git', '').toLowerCase()
                    env.APP_NAME = repoName
                    
                    echo "--- Iniciando Pipeline: ${env.APP_NAME} ---"
                    echo "Rama: ${params.BRANCH}"
                    echo "Tecnología: ${params.LANGUAGE}"
                    
                    cleanWs() // Limpiar espacio de trabajo anterior
                    
                    // Clonar código fuente y herramientas en paralelo (secuencial en script)
                    dir('app') { 
                        git branch: params.BRANCH, url: params.REPO_URL 
                    }
                    dir('tooling') { 
                        git branch: 'main', url: env.TEMPLATES_REPO 
                    }
                }
            }
        }

        stage('2. Construcción (Build)') {
            steps {
                script {
                    echo "--- Construyendo Imagen Docker ---"
                    
                    // Copia el Dockerfile correcto según la selección del usuario (node/angular/java)
                    sh "cp tooling/docker/${params.LANGUAGE}/Dockerfile app/Dockerfile"
                    
                    dir('app') {
                        // --network host ayuda a que npm install no falle por red
                        sh "docker build --network host -t ${env.APP_NAME}:${env.BUILD_NUMBER} ."
                    }
                }
            }
        }

        stage('3. Despliegue (Deploy)') {
            steps {
                script {
                    echo "--- Desplegando en puerto ${params.HOST_PORT} ---"
                    
                    withEnv([
                        "APP_IMAGE=${env.APP_NAME}:${env.BUILD_NUMBER}",
                        "APP_PORT=${params.HOST_PORT}", 
                        "ENVIRONMENT=${params.BRANCH}",
                        // Variables de BD (Hardcoded para MVP, idealmente usar Credenciales de Jenkins)
                        "DB_HOST=postgres_db", 
                        "DB_NAME=mirai_db",
                        "DB_USER=postgres",
                        "DB_PASS=donlito123"
                    ]) {
                        dir('tooling/templates') {
                            // Descargar docker-compose si no existe (o usar el del sistema)
                            sh "curl -SL https://github.com/docker/compose/releases/download/v2.29.1/docker-compose-linux-x86_64 -o docker-compose"
                            sh "chmod +x docker-compose"

                            // Lógica de limpieza de puertos (Matar contenedor viejo si estorba)
                            sh """
                                CONFLICT_ID=\$(docker ps -q --filter publish=${params.HOST_PORT})
                                if [ ! -z "\$CONFLICT_ID" ]; then
                                    echo "El puerto ${params.HOST_PORT} está ocupado. Liberando..."
                                    docker rm -f \$CONFLICT_ID
                                else
                                    echo "Puerto ${params.HOST_PORT} libre."
                                fi
                            """

                            // Reiniciar servicio
                            sh "./docker-compose -p ${env.APP_NAME} -f docker-compose.yml down || true"
                            sh "./docker-compose -p ${env.APP_NAME} -f docker-compose.yml up -d"
                            
                            // Verificación rápida de salud
                            sh "sleep 5" 
                            sh "docker ps | grep ${env.APP_NAME}"
                        }
                    }
                }
            }
        }
    }

    // SECCIÓN POST: Se ejecuta al finalizar, pase lo que pase
    post {
        success {
            script {
                echo "Pipeline Exitoso. Enviando correo..."
                mail(
                    to: "${params.EMAIL_TO}",
                    subject: "Despliegue Exitoso: ${env.APP_NAME} v${env.BUILD_NUMBER}",
                    mimeType: 'text/html',
                    body: """
                        <div style="font-family: Arial, sans-serif; padding: 20px; border: 1px solid #e0e0e0; border-radius: 8px; background-color: #f9fafb;">
                            <h2 style="color: #4f46e5;">Zalgodyne IDP - Despliegue Completado</h2>
                            <p>Tu aplicación <b>${env.APP_NAME}</b> se ha desplegado correctamente y está lista para usarse.</p>
                            
                            <div style="background-color: white; padding: 15px; border-radius: 5px; border: 1px solid #ddd;">
                                <ul style="list-style: none; padding: 0;">
                                    <li><b>Repositorio:</b> ${params.REPO_URL}</li>
                                    <li><b>Rama:</b> <span style="background-color: #e0e7ff; color: #4338ca; padding: 2px 6px; border-radius: 4px;">${params.BRANCH}</span></li>
                                    <li><b>Tecnología:</b> ${params.LANGUAGE}</li>
                                    <li><b>Build ID:</b> #${env.BUILD_NUMBER}</li>
                                </ul>
                            </div>
                            
                            <br>
                            <a href="http://localhost:${params.HOST_PORT}" style="background-color: #10b981; color: white; padding: 10px 20px; text-decoration: none; border-radius: 5px; font-weight: bold;">
                                Ver Aplicación (: ${params.HOST_PORT})
                            </a>
                            <p style="font-size: 12px; color: #888; margin-top: 20px;">Zalgodyne Internal Developer Platform</p>
                        </div>
                    """
                )
            }
        }
        failure {
            script {
                echo "Pipeline Fallido. Enviando alerta..."
                mail(
                    to: "${params.EMAIL_TO}",
                    subject: "Falló el Despliegue: ${env.APP_NAME} v${env.BUILD_NUMBER}",
                    mimeType: 'text/html',
                    body: """
                        <div style="font-family: Arial, sans-serif; padding: 20px; border: 1px solid #fee2e2; border-radius: 8px; background-color: #fef2f2;">
                            <h2 style="color: #ef4444;">Error en el Despliegue</h2>
                            <p>Hubo un problema procesando el proyecto <b>${env.APP_NAME}</b>.</p>
                            <p>Por favor, revisa los logs en Jenkins para ver el detalle del error.</p>
                            <ul>
                                <li><b>Rama:</b> ${params.BRANCH}</li>
                                <li><b>Build ID:</b> #${env.BUILD_NUMBER}</li>
                            </ul>
                        </div>
                    """
                )
            }
        }
    }
}