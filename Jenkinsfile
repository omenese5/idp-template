pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'URL del repositorio Cliente')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Rama a desplegar')
        string(name: 'LANGUAGE', defaultValue: 'java', description: 'Lenguaje (java/node)')
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

        stage('2. Construcción (Build Real)') {
            steps {
                script {
                    echo "Preparando construcción para: ${params.LANGUAGE}"

                    sh "cp tooling/docker/${params.LANGUAGE}/Dockerfile app/Dockerfile"

                    dir('app') {
                        echo "Iniciando Docker Build (Esto puede tardar la primera vez)..."
                        
                        sh "docker build --network host -t ${APP_NAME}:${env.BUILD_NUMBER} ."
                    }
                    echo "Imagen ${APP_NAME}:${env.BUILD_NUMBER} construida exitosamente."
                }
            }
        }

        stage('3. Despliegue') {
            steps {
                script {
                    echo "Desplegando aplicación..."
                    
                    withEnv([
                        "APP_IMAGE=${APP_NAME}:${env.BUILD_NUMBER}",
                        "APP_PORT=8085",
                        "ENVIRONMENT=prod",
                        "DB_HOST=postgres_db", 
                        "DB_NAME=mirai_db",
                        "DB_USER=postgres",
                        "DB_PASS=donlito123"
                    ]) {
                        dir('tooling/templates') {
                            echo "Configurando Docker Compose Standalone..."
                    
                            sh "curl -SL https://github.com/docker/compose/releases/download/v2.29.1/docker-compose-linux-x86_64 -o docker-compose"
                            sh "chmod +x docker-compose"

                            echo "Ejecutando despliegue..."
                            sh "./docker-compose -f docker-compose.yml down || true"
                            sh "./docker-compose -f docker-compose.yml up -d"
                        }
                    }
                    
                    echo "App desplegada correctamente."
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
                            body: "<h1>IDP Platform</h1><p>El proyecto <b>${params.REPO_URL}</b> está activo y construido.</p>"
                        )
                    } catch (e) {
                        echo "Notificación simulada (SMTP no configurado)."
                    }
                }
            }
        }
    }
}