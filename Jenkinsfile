pipeline {
    agent any

    environment {
        COLLECTION = 'Collection/ACHDATA - YY.postman_collection.json'
        ENVIRONMENT = 'Environment/ACHData QA.postman_environment.json'
        DATA1 = 'File/Incluir_Excluir Personas.csv'
        DATA2 = 'File/50 Registros.csv'
        DATA3 = 'File/cargue_masivo_usuarios.csv'

        REPORT_BASE = "Ejecuciones/ACHDATA"
        EMAIL_TO = "yeinerballesta@cbit-online.com"
        SMTP_SERVER = "email.periferia-it.com"
        SMTP_PORT = "587"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validar herramientas') {
            steps {
                powershell '''
                    Write-Host "Node:"
                    node -v
                    Write-Host "Newman:"
                    newman -v
                '''
            }
        }

        stage('Ejecutar pruebas') {
            steps {
                powershell '''
                    $fecha = Get-Date -Format "yyyy-MM-dd"
                    $hora = Get-Date -Format "HH-mm"

                    $global:REPORT_PATH = "$env:REPORT_BASE/Fecha_$fecha`_Hora_$hora"

                    New-Item -ItemType Directory -Force -Path $global:REPORT_PATH | Out-Null

                    Write-Host "Ruta de ejecución: $global:REPORT_PATH"

                    newman run "$env:COLLECTION" `
                        -e "$env:ENVIRONMENT" `
                        --insecure `
                        --iteration-data "$env:DATA1" `
                        --iteration-data "$env:DATA2" `
                        --iteration-data "$env:DATA3" `
                        --reporters cli,htmlextra,json `
                        --reporter-htmlextra-export "$global:REPORT_PATH/ReporteGeneral.html" `
                        --reporter-json-export "$global:REPORT_PATH/resultado.json"
                '''
            }
        }

        stage('Procesar métricas por carpeta') {
            steps {
                powershell '''
                    $json = Get-Content "$global:REPORT_PATH/resultado.json" | ConvertFrom-Json

                    $folders = @("AT","SS","CS","CER","CERCS")
                    $summary = @()

                    foreach ($folder in $folders) {

                        $tests = $json.run.executions | Where-Object {
                            $_.item.name -match $folder
                        }

                        $total = $tests.Count
                        $passed = ($tests | Where-Object { $_.assertions.failed -eq $false }).Count
                        $failed = $total - $passed

                        $summary += [PSCustomObject]@{
                            Folder = "ACHDATA - $folder"
                            Total = $total
                            Exitosos = $passed
                            Fallidos = $failed
                        }
                    }

                    $summary | Export-Csv "$global:REPORT_PATH/resumen.csv" -NoTypeInformation -Encoding UTF8
                    $summary | Format-Table -AutoSize

                    $global:MAIL_BODY = $summary | Out-String
                '''
            }
        }

        stage('Enviar correo') {
            steps {
                powershell '''
                    $smtp = "$env:SMTP_SERVER"
                    $port = $env:SMTP_PORT
                    $to = "$env:EMAIL_TO"
                    $from = "jenkins@periferia-it.com"

                    $subject = "Reporte Newman - ACHDATA - $(Get-Date -Format 'yyyy-MM-dd HH:mm')"

                    $body = @"
Ejecución Newman ACHDATA

Resumen por carpeta:

$global:MAIL_BODY

Ruta de ejecución:
$global:REPORT_PATH
"@

                    $attachments = Get-ChildItem "$global:REPORT_PATH" -Filter *.html | Select-Object -Expand FullName

                    Send-MailMessage `
                        -To $to `
                        -From $from `
                        -Subject $subject `
                        -Body $body `
                        -SmtpServer $smtp `
                        -Port $port `
                        -Attachments $attachments
                '''
            }
        }
    }

    post {
        success {
            echo 'Ejecución Newman finalizada correctamente y correo enviado.'
        }
        failure {
            echo 'Fallaron las pruebas o el envío de correo.'
        }
    }
}