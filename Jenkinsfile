pipeline {
    agent any

    environment {
        COLLECTION   = "Collection\\ACHDATA - YY.postman_collection.json"
        ENVIRONMENT  = "Environment\\ACHData QA.postman_environment.json"
        REPORTS      = "reports"
        FECHA        = new Date().format("yyyy-MM-dd")
        HORA         = new Date().format("HHmmss")
        TS           = "${FECHA}_${HORA}"

        CSV_INCLUIR  = "File\\Incluir_Excluir Personas.csv"
        CSV_50       = "File\\50 Registros.csv"
        CSV_MASIVO   = "File\\cargue_masivo_usuarios.csv"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Instalar Newman') {
            steps {
                powershell '''
                if (!(Get-Command newman -ErrorAction SilentlyContinue)) {
                    npm install -g newman
                    npm install -g newman-reporter-htmlextra
                }
                '''
            }
        }

        stage('Crear carpeta reportes') {
            steps {
                powershell '''
                if (!(Test-Path $env:REPORTS)) {
                    New-Item -ItemType Directory -Path $env:REPORTS
                }
                '''
            }
        }

        stage('Ejecutar Colección por Carpetas') {
            steps {
                script {

                    def folders = ['AT','SS','CS','CER','CERCS']

                    for (folder in folders) {

                        powershell """
                        Write-Host "Ejecutando carpeta: ${folder}" -ForegroundColor Cyan

                        newman run "${COLLECTION}" `
                          --folder "${folder}" `
                          --environment "${ENVIRONMENT}" `
                          --env-var "fileIncluirExcluir=${CSV_INCLUIR}" `
                          --env-var "file50Registros=${CSV_50}" `
                          --env-var "fileCargueMasivo=${CSV_MASIVO}" `
                          --delay-request 1000 `
                          --timeout-request 30000 `
                          --reporters "cli,htmlextra,junit" `
                          --reporter-htmlextra-export "${REPORTS}\\ACHDATA_${folder}_${TS}.html" `
                          --reporter-junit-export "${REPORTS}\\ACHDATA_${folder}_${TS}.xml" `
                          --reporter-htmlextra-title "ACHDATA ${folder} - ${FECHA} ${HORA}" `
                          --insecure `
                          --verbose
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: 'reports/*.xml'
            archiveArtifacts artifacts: 'reports/*.html', fingerprint: true
        }
    }
}