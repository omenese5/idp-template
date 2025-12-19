pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'URL del repositorio Cliente')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Rama a desplegar')
        string(name: 'LANGUAGE', defaultValue: 'node', description: 'Lenguaje')
        string(name: 'EMAIL_TO', defaultValue: '', description: 'Email')
    }

    environment {
        APP_NAME = "idp-app"
        TEMPLATES_REPO = "https://github.com/omenese5/idp-template.git"
    }

    stages {
        stage('1. Preparar Workspace') {
            steps {
                script {
                    echo "Limpiando espacio de trabajo..."
                    cleanWs()
                    
                    echo "Descargando aplicación del cliente..."
                    dir('app') {
                        git branch: params.BRANCH, url: params.REPO_URL
                    }

                    echo "Descargando herramientas de plataforma..."
                    dir('tooling') {
                        git branch: 'main', url: env.TEMPLATES_REPO
                    }
                }
            }
        }

        stage('2. Construcción (Build)') {
            steps {
                script {
                    echo "Construyendo imagen Docker para: ${params.LANGUAGE}"

                    dir('app') {
                        echo "Docker Build (Simulado)..."
                        sleep 2
                    }
                    echo "Imagen construida."
                }
            }
        }

        stage('3. Despliegue (Deploy)') {
            steps {
                script {
                    echo "Desplegando con Docker Compose..."
                    
                    withEnv([
                        "APP_IMAGE=${APP_NAME}:${env.BUILD_NUMBER}",
                        "APP_PORT=3005",
                        "ENVIRONMENT=demo"
                    ]) {
                        dir('tooling/templates') {
                            echo "Usando plantilla: compose-deploy.yml"
                            
                            sh "docker compose -f compose-deploy.yml down || true"
                            sh "docker compose -f compose-deploy.yml up -d"
                        }
                    }
                    echo "Servicio desplegado en puerto 3005."
                }
            }
        }

        stage('4. Notificación') {
            steps {
                script {
                    echo "Notificando a: ${params.EMAIL_TO}"
                    try {
                        mail(
                            to: "${params.EMAIL_TO}",
                            subject: "Despliegue IDP Exitoso #${env.BUILD_NUMBER}",
                            mimeType: 'text/html',
                            body: "<h1>Zalgodyne IDP</h1><p>El proyecto <b>${params.REPO_URL}</b> está activo.</p>"
                        )
                    } catch (e) {
                        echo "Notificación simulada (SMTP no configurado)."
                    }
                }
            }
        }
    }
}