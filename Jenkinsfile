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
                    echo "Desplegando ${env.APP_NAME} en puerto ${params.HOST_PORT}..."
                    
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
                            sh "chmod +x docker-compose"
                            
                            sh """
                                CONFLICT_ID=\$(docker ps -q --filter publish=${params.HOST_PORT})
                                if [ ! -z "\$CONFLICT_ID" ]; then
                                    echo "El puerto ${params.HOST_PORT} está ocupado por \$CONFLICT_ID. Liberando..."
                                    docker rm -f \$CONFLICT_ID
                                else
                                    echo "Puerto ${params.HOST_PORT} libre."
                                fi
                            """

                            sh "./docker-compose -p ${env.APP_NAME} -f docker-compose.yml down || true"
                            sh "./docker-compose -p ${env.APP_NAME} -f docker-compose.yml up -d"
                        }
                    }
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
                            subject: "Despliegue Exitoso: ${env.APP_NAME} #${env.BUILD_NUMBER}",
                            mimeType: 'text/html',
                            body: """
                                <h1>Zalgodyne IDP</h1>
                                <p>El proyecto <b>${env.APP_NAME}</b> se ha desplegado correctamente.</p>
                                <ul>
                                    <li><b>Repositorio:</b> ${params.REPO_URL}</li>
                                    <li><b>Ambiente/Rama:</b> ${params.BRANCH}</li>
                                    <li><b>Build ID:</b> #${env.BUILD_NUMBER}</li>
                                </ul>
                            """
                        )
                    } catch (e) {
                        echo "Notificación simulada (SMTP no configurado)."
                    }
                }
            }
        }
    }
}