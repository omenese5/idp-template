pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: '', description: 'URL del repositorio Cliente')
        string(name: 'BRANCH', defaultValue: 'main', description: 'Rama a desplegar')
        string(name: 'LANGUAGE', defaultValue: 'java', description: 'Lenguaje (java/node)')
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
                    echo "Puerto: ${params.HOST_PORT}"
                    
                    cleanWs()
                    dir('app') { git branch: params.BRANCH, url: params.REPO_URL }
                    dir('tooling') { git branch: 'main', url: env.TEMPLATES_REPO }
                }
            }
        }

        stage('2. Construcción') {
            steps {
                script {
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
                            echo "Configurando Docker Compose..."
                            
                            sh "curl -SL https://github.com/docker/compose/releases/download/v2.29.1/docker-compose-linux-x86_64 -o docker-compose"
                            
                            sh "chmod +x docker-compose"

                            echo "--- LIMPIEZA DE PUERTOS ---"
                            sh """
                                CONFLICT_ID=\$(docker ps -q --filter publish=${params.HOST_PORT})
                                if [ ! -z "\$CONFLICT_ID" ]; then
                                    echo "El puerto ${params.HOST_PORT} está ocupado por \$CONFLICT_ID. Liberando..."
                                    docker rm -f \$CONFLICT_ID
                                else
                                    echo "Puerto ${params.HOST_PORT} libre."
                                fi
                            """

                            echo "Ejecutando despliegue..."
                            sh "./docker-compose -p ${env.APP_NAME} -f docker-compose.yml down || true"
                            sh "./docker-compose -p ${env.APP_NAME} -f docker-compose.yml up -d"
                        }
                    }
                    
                    echo "App desplegada correctamente."
                }
            }
        }

        stage('4. Verificación (Health Check)') {
            steps {
                script {
                    echo "Iniciando Health Check en puerto ${params.HOST_PORT}..."
                    
                    sleep 15 

                    echo "Probando conectividad HTTP..."
                    
                    sh """
                        curl --fail \
                             --retry 5 \
                             --retry-connrefused \
                             --retry-delay 5 \
                             --max-time 10 \
                             http://localhost:${params.HOST_PORT}/
                    """
                    
                    echo "La aplicación está viva y respondiendo."
                }
            }
        }
    }
}