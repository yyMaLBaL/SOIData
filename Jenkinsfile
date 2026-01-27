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
                    newman run postman/collections/ACHDATA - YY.postman_collection.json `
                        -e postman/environments/ACHData QA.postman_environment.json `
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