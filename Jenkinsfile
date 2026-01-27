pipeline {
    agent any

    parameters {
        choice(name: 'AMBIENTE', choices: ['ALL','AT','SS','CS','CER','CERCS'], description: 'Ambiente')
        choice(name: 'SECCION', choices: ['TODAS','Autenticacion','Consultas','Servicios ACH','SDH','Administracion','Monitoreo'], description: 'Seccion')
    }

    environment {
        SMTP_SERVER = 'email.periferia-it.com'
        SMTP_PORT   = '587'
        EMAIL_TO    = 'yeinerballesta@cbit-online.com'
    }

    stages {

        stage('Checkout') { steps { checkout scm } }

        stage('Validar Newman') {
            steps { powershell 'node -v; newman -v' }
        }

        stage('Configurar Ejecucion') {
            steps {
                script {
                    def dt = powershell(returnStdout: true, script: 'Get-Date -Format "yyyy-MM-dd_HH-mm-ss"').trim()
                    env.EXECUTION_FOLDER = "Ejecuciones/ACHDATA/${dt}"
                    env.EXECUTION_DATE = dt.split('_')[0]
                    env.EXECUTION_TIME = dt.split('_')[1]
                    powershell "New-Item -ItemType Directory -Force -Path '${env.EXECUTION_FOLDER}'"
                }
            }
        }

        stage('Ejecutar en Paralelo') {
            steps {
                script {

                    def secciones = [
                        'Autenticacion':['Autenticación'],
                        'Consultas':[
                            'Detallada Natural Y Jurídica','Consolidada Natural','Consolidada Jurídica',
                            'Extendida Natural','Masivas Con Filtros','Transferencias Y Pagos ACH Personas',
                            'Transferencias Y Pagos ACH Empresas','Contactabilidad Natural Y Jurídica',
                            'Parámetros Consultas','Prueba De Cobertura',
                            'Consulta por categoría - Persona Natural','Otras Consultas Masivas'
                        ],
                        'Servicios ACH':['Servicios ACH'],
                        'SDH':['SDH'],
                        'Administracion':['Estadísticas','Auditoría','Usuarios','Roles','Carga Masiva de Usuarios'],
                        'Monitoreo':['Monitoreo']
                    ]

                    def carpetas = params.SECCION == 'TODAS' ? secciones.values().flatten() : secciones[params.SECCION]
                    def ambientes = params.AMBIENTE == 'ALL' ? ['AT','SS','CS','CER','CERCS'] : [params.AMBIENTE]

                    def resumenGlobal = [:]
                    ambientes.each { resumenGlobal[it] = [total:0, ok:0, fail:0] }

                    def ejecuciones = [:]

                    ambientes.each { amb ->
                        ejecuciones[amb] = {
                            carpetas.each { carpeta ->

                                powershell """
                                    \$safe = ("${amb}_${carpeta}" -replace ' ', '_' -replace '[^a-zA-Z0-9_-]', '')
                                    \$html = "${env.EXECUTION_FOLDER}\\report_\$safe.html"
                                    \$json = "${env.EXECUTION_FOLDER}\\report_\$safe.json"

                                    \$out = newman run "Collection/ACHDATA - YY.postman_collection.json" `
                                        -e "Environment/ACHData QA.postman_environment.json" `
                                        --folder "${amb} - ${carpeta}" `
                                        --insecure `
                                        --reporters cli,html,json `
                                        --reporter-html-export "\$html" `
                                        --reporter-json-export "\$json" 2>&1 | Out-String

                                    if (Test-Path "\$json") {
                                        \$j = Get-Content "\$json" -Raw | ConvertFrom-Json
                                        Write-Output "##${amb}##|\$((\$j.run.stats.requests.total))|\$((\$j.run.stats.requests.total - \$j.run.stats.requests.failed))|\$((\$j.run.stats.requests.failed))"
                                    }
                                """
                            }
                        }
                    }

                    def salida = parallel ejecuciones

                }
            }
        }

        stage('Enviar Correo') {
            steps {
                script {

                    def htmlFiles = powershell(returnStdout: true, script: """
                        Get-ChildItem '${env.EXECUTION_FOLDER}' -Filter *.html | Measure-Object | Select -ExpandProperty Count
                    """).trim()

                    def emailBody = """
                    <html><body style="font-family:Arial">
                    <h2>Reporte Postman ACHDATA</h2>
                    <p><b>Fecha:</b> ${env.EXECUTION_DATE}</p>
                    <p><b>Hora:</b> ${env.EXECUTION_TIME}</p>

                    <p><b>Ambientes ejecutados:</b> ${params.AMBIENTE}</p>
                    <p><b>Sección:</b> ${params.SECCION}</p>

                    <p>Total de reportes HTML generados: <b>${htmlFiles}</b></p>

                    <p>Se adjuntan únicamente los reportes HTML.</p>
                    </body></html>
                    """

                    emailext(
                        to: EMAIL_TO,
                        subject: "Reporte Postman ACHDATA - ${env.EXECUTION_DATE}",
                        mimeType: 'text/html',
                        attachmentsPattern: "${env.EXECUTION_FOLDER}/**/*.html",
                        body: emailBody
                    )
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: "${env.EXECUTION_FOLDER}/**/*", allowEmptyArchive: true
        }
    }
}
