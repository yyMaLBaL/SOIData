pipeline {
    agent any

    environment {
        ENV_FILE = 'postman/environments/qa.postman_environment.json'
        COLLECTION_PATH = 'postman/collections'
    }

    stages {

        stage('Checkout') {
            steps {
                git url: 'https://github.com/yyMaLBaL/SOIData.git',
                    branch: 'main'
            }
        }

        stage('Run Postman Collections') {
            steps {
                sh '''
                    set -e

                    echo "===== Ejecutando Colección: ACHData ====="
                    newman run $COLLECTION_PATH/ACHData.postman_collection.json \
                        -e $ENV_FILE \
                        --reporters cli

                    echo "===== Ejecutando Colección: SOIData ====="
                    newman run $COLLECTION_PATH/SOIData.postman_collection.json \
                        -e $ENV_FILE \
                        --reporters cli

                    echo "===== Ejecutando Colección: Transferencias ====="
                    newman run $COLLECTION_PATH/Transferencias.postman_collection.json \
                        -e $ENV_FILE \
                        --reporters cli
                '''
            }
        }
    }

    post {
        success {
            echo 'Todas las colecciones Postman se ejecutaron correctamente'
        }
        failure {
            echo 'Una o más colecciones Postman fallaron'
        }
    }
}
