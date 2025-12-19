pipeline {
    agent any

    parameters {
        string(name: 'REPO_URL', defaultValue: 'https://github.com/omenese5/Mirai-Api.git', description: 'URL del repositorio')
        string(name: 'BRANCH', defaultValue: 'development', description: 'Branch a construir')
        choice(name: 'LANGUAGE', choices: ['java', 'node'], description: 'Selecciona el lenguaje del proyecto')
        choice(name: 'ENVIRONMENT', choices: ['QA', 'PROD'], description: 'Entorno de Despliegue')
        string(name: 'EMAIL_TO', defaultValue: 'luis.meneses.arirama@gmail.com', description: 'Correo para notificaciones')
    }

    environment {
        GIT_CREDENTIALS = 'github-token'
        IMAGE_NAME      = "zalgodyne/${params.LANGUAGE-app}" 
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('1. Checkout') {
            steps {
                script {
                    echo "Clonando rama ${params.BRANCH}..."
                    git branch: params.BRANCH, credentialsId: env.GIT_CREDENTIALS, url: params.REPO_URL
                }
            }
        }

        stage('2. Code Analysis & Unit Tests') {
            steps {
                script {
                    echo "Ejecutando pruebas y análisis para ${params.LANGUAGE}..."
                    if (params.LANGUAGE == 'java') {
                        sh 'chmod +x mvnw'
                        sh './mvnw test -DskipITs' 
                    } else {
                        sh 'npm install'
                        sh 'npm run test' 
                    }
                }
            }
        }

        stage('3. Docker Build (CI)') {
            steps {
                script {
                    echo "Construyendo imagen Docker..."
                    def dockerfile = "docker/${params.LANGUAGE}/Dockerfile"
                    sh "docker build -t ${env.IMAGE_NAME}:${env.IMAGE_TAG} -f ${dockerfile} ."

                }
            }
        }

        stage('4. Deploy (CD)') {
            steps {
                script {
                    echo "Desplegando en entorno: ${params.ENVIRONMENT}..."
                    withEnv(["APP_IMAGE=${env.IMAGE_NAME}:${env.IMAGE_TAG}", "APP_PORT=8080"]) {
                        dir('deploy') { 
                            sh 'docker-compose up -d --force-recreate'
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            script {
                enviarCorreo("Exitoso", "#28a745", "#d4edda", "#155724")
            }
        }
        failure {
            script {
                enviarCorreo("Fallido", "#dc3545", "#f8d7da", "#721c24")
            }
        }
    }
}

// --- CORREO EMAIL ---

def enviarCorreo(estado, colorHeader, colorBg, colorText) {
    mail(
        to: "${params.EMAIL_TO}",
        subject: "Pipeline ${estado}: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        body: """
            <!DOCTYPE html>
            <html lang="es">
            <head>
              <meta charset="UTF-8">
              <style>
                body { font-family: Arial, sans-serif; background-color: #f7f8fa; margin:0; padding:0; }
                .container { max-width:600px; margin:20px auto; background-color:#ffffff; border-radius:10px; box-shadow:0 4px 10px rgba(0,0,0,0.1); overflow:hidden; }
                .header { background-color:${colorHeader}; color:white; text-align:center; padding:20px; }
                .content { padding:20px; text-align:center; }
                .status { display:inline-block; background-color:${colorBg}; color:${colorText}; font-weight:bold; padding:10px 20px; border-radius:8px; margin: 20px 0;}
              </style>
            </head>
            <body>
              <div class="container">
                <div class="header"><h1>${env.JOB_NAME}</h1></div>
                <div class="content">
                  <h2>Pipeline ${estado}</h2>
                  <p><b>Branch:</b> ${params.BRANCH} | <b>Env:</b> ${params.ENVIRONMENT}</p>
                  <p class="status">${estado}</p>
                  <p><a href="${env.BUILD_URL}">Ver Logs en Jenkins</a></p>
                </div>
              </div>
            </body>
            </html>
        """,
        mimeType: 'text/html'
    )
}