pipeline {
    agent any

    environment {
        FECHA = powershell(script: "Get-Date -Format yyyy-MM-dd", returnStdout: true).trim()
        HORA  = powershell(script: "Get-Date -Format HH-mm", returnStdout: true).trim()
        TS    = powershell(script: "Get-Date -Format yyyyMMdd_HHmm", returnStdout: true).trim()
        BASE_REPORT = "Ejecuciones\\ACHDATA\\Fecha_${FECHA}Hora${HORA}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Crear estructura') {
            steps {
                powershell '''
                New-Item -ItemType Directory -Force -Path "$env:BASE_REPORT" | Out-Null
                '''
            }
        }

        stage('Ejecutar AT') {
            steps {
                powershell '''
                $fileIncluirExcluir = Join-Path $env:WORKSPACE "File\\Incluir_Excluir Personas.csv"
                $file50Registros    = Join-Path $env:WORKSPACE "File\\50 Registros.csv"
                $fileCargueMasivo   = Join-Path $env:WORKSPACE "File\\cargue_masivo_usuarios.csv"

                Write-Host "Iniciando ejecución AT" -ForegroundColor Cyan

                newman run "Collection\\ACHDATA - AT.postman_collection.json" `
                  --environment "Environment\\ACHData QA.postman_environment.json" `
                  --env-var "fileIncluirExcluir=$fileIncluirExcluir" `
                  --env-var "file50Registros=$file50Registros" `
                  --env-var "fileCargueMasivo=$fileCargueMasivo" `
                  --delay-request 1000 `
                  --timeout-request 31000 `
                  --reporters "cli,htmlextra,junit" `
                  --reporter-htmlextra-export "$env:BASE_REPORT\\ACHData_AT_Report_$env:TS.html" `
                  --reporter-junit-export "$env:BASE_REPORT\\ACHData_AT_Report_$env:TS.xml" `
                  --reporter-htmlextra-title "ACHData AT - $env:FECHA $env:HORA" `
                  --insecure `
                  --verbose
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'Ejecuciones/**', fingerprint: true
        }
        failure {
            echo 'Fallaron las pruebas'
        }
        success {
            echo ' Ejecución AT completada correctamente'
        }
    }
}