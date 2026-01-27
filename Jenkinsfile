pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validar Newman') {
            steps {
                powershell '''
                    Write-Host "Verificando version de Node..."
                    node -v

                    Write-Host "Verificando version de Newman..."
                    newman -v
                '''
            }
        }

        stage('Run Postman Tests') {
            steps {
                powershell '''
                    newman run Colecciones/"ACHDATA - YY.postman_collection.json" `
                        -e Entornos/"ACHData QA.postman_environment.json" `
                        --reporters cli
                '''
            }
        }
    }

    post {
        failure {
            echo 'Fallaron las pruebas Postman'
        }
        success {
            echo 'Ejecucion completada correctamente'
        }
    }
}