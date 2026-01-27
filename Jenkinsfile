pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                git url: 'https://github.com/yyMaLBaL/SOIData.git',
                    branch: 'main'
            }
        }

        stage('Run Postman Tests') {
            steps {
                sh '''
                newman run postman/collections/Cars_API.postman_collection.json \
                -e postman/environments/qa.postman_environment.json \
                --reporters cli
                '''
            }
        }
    }

    post {
        success {
            echo 'Pruebas Postman ejecutadas correctamente'
        }
        failure {
            echo 'Fallaron las pruebas Postman'
        }
    }
}