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
        // --- VISUAL STAGE 1: SOURCE ---
        stage('1. Preparar Workspace') {
            steps {
                script {
                    // ETIQUETA PARA EL FRONTEND
                    echo "--- [STAGE: SOURCE] ---"
                    
                    def urlParts = params.REPO_URL.tokenize('/')
                    def repoName = urlParts.last().replace('.git', '').toLowerCase()
                    env.APP_NAME = repoName
                    
                    echo "App: ${env.APP_NAME}"
                    cleanWs()
                    dir('app') { git branch: params.BRANCH, url: params.REPO_URL }
                    dir('tooling') { git branch: 'main', url: env.TEMPLATES_REPO }
                }
            }
        }

        // --- VISUAL STAGE 2: BUILD ---
        stage('2. Preparación (Build)') {
            steps {
                script {
                    // ETIQUETA PARA EL FRONTEND
                    echo "--- [STAGE: BUILD] ---"
                    
                    // Aquí movemos los archivos necesarios antes de dockerizar
                    echo "Preparando Dockerfile para ${params.LANGUAGE}..."
                    sh "cp tooling/docker/${params.LANGUAGE}/Dockerfile app/Dockerfile"
                }
            }
        }

        // --- VISUAL STAGE 3: DOCKERIZE ---
        stage('3. Construir Imagen (Dockerize)') {
            steps {
                script {
                    // ETIQUETA PARA EL FRONTEND
                    echo "--- [STAGE: DOCKERIZE] ---"
                    
                    dir('app') {
                        echo "Construyendo imagen Docker..."
                        sh "docker build --network host -t ${env.APP_NAME}:${env.BUILD_NUMBER} ."
                    }
                }
            }
        }

        // --- VISUAL STAGE 4: DEPLOY (Incluye Email) ---
        stage('4. Despliegue y Notificación') {
            steps {
                script {
                    // ETIQUETA PARA EL FRONTEND
                    echo "--- [STAGE: DEPLOY] ---"
                    
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
                            // 1. Lógica de Docker Compose
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
                            
                            sh "sleep 5" // Espera técnica para que levante
                            
                            // 2. Lógica de Email (INTERNA EN ESTE STAGE)
                            // El usuario sigue viendo "Deploy" mientras esto ocurre
                            echo "--- Enviando Notificación Interna ---"
                            try {
                                mail(
                                    to: "${params.EMAIL_TO}",
                                    subject: "Despliegue Exitoso: ${env.APP_NAME} v${env.BUILD_NUMBER}",
                                    mimeType: 'text/html',
                                    body: """
                                        <div style="font-family: Arial, sans-serif; padding: 20px; border: 1px solid #e0e0e0;">
                                            <h2 style="color: #2da44e;">Zalgodyne IDP - Online</h2>
                                            <p>El proyecto <b>${env.APP_NAME}</b> se ha desplegado correctamente.</p>
                                            <ul>
                                                <li><b>Puerto:</b> ${params.HOST_PORT}</li>
                                                <li><b>Build ID:</b> #${env.BUILD_NUMBER}</li>
                                            </ul>
                                        </div>
                                    """
                                )
                                echo "Email enviado correctamente."
                            } catch (e) {
                                echo "Advertencia: El despliegue funcionó pero falló el email: ${e.message}"
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        failure {
            script {
                mail(
                    to: "${params.EMAIL_TO}",
                    subject: "Falló el Despliegue: ${env.APP_NAME}",
                    mimeType: 'text/html',
                    body: "<p>Hubo un error crítico en el pipeline. Revisa Jenkins.</p>"
                )
            }
        }
    }
}