pipeline {
    agent any
    
    parameters {
        choice(
            name: 'COLLECTION_FOLDER',
            choices: ['AT', 'SS', 'CS', 'CER', 'CERCS'],
            description: 'Selecciona la carpeta a ejecutar'
        )
    }
    
    environment {
        COLLECTION_PATH = "Collection/\"ACHDATA - YY.postman_collection.json\""
        ENVIRONMENT_PATH = "Environment/\"ACHData QA.postman_environment.json\""
        DATA_FILE_1 = "File/\"Incluir_Excluir Personas.csv\""
        DATA_FILE_2 = "File/\"50 Registros.csv\""
        DATA_FILE_3 = "File/\"cargue_masivo_usuarios.csv\""
        EXECUTION_DATE = sh(script: 'date +"%Y-%m-%d"', returnStdout: true).trim()
        EXECUTION_TIME = sh(script: 'date +"%H-%M-%S"', returnStdout: true).trim()
        EXECUTION_FOLDER = "Ejecuciones/ACHDATA/Fecha_${EXECUTION_DATE}_Hora_${EXECUTION_TIME}"
        HTML_REPORT = "${EXECUTION_FOLDER}/report.html"
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
        
        stage('Crear Carpeta de Ejecución') {
            steps {
                powershell '''
                    Write-Host "Creando carpeta de ejecución: ${env.EXECUTION_FOLDER}"
                    New-Item -ItemType Directory -Force -Path "${env.EXECUTION_FOLDER}"
                '''
            }
        }
        
        stage('Ejecutar Colección Postman') {
            steps {
                script {
                    def folders = params.COLLECTION_FOLDER == 'ALL' ? 
                        ['AT', 'SS', 'CS', 'CER', 'CERCS'] : 
                        [params.COLLECTION_FOLDER]
                    
                    def executionResults = [:]
                    
                    folders.each { folder ->
                        stage("Ejecutar ${folder}") {
                            powershell """
                                \$folderName = "${folder}"
                                \$reportFile = "${env.EXECUTION_FOLDER}/report_\${folderName}.html"
                                \$jsonReport = "${env.EXECUTION_FOLDER}/report_\${folderName}.json"
                                
                                Write-Host "Ejecutando colección para carpeta: \${folderName}"
                                
                                # Ejecutar Newman con reporte HTML y JSON
                                newman run ${env.COLLECTION_PATH} `
                                    -e ${env.ENVIRONMENT_PATH} `
                                    --folder "\${folderName}" `
                                    --insecure `
                                    --reporters cli,html,json `
                                    --reporter-html-export "\${reportFile}" `
                                    --reporter-json-export "\${jsonReport}" `
                                    --iteration-data ${env.DATA_FILE_1} `
                                    --iteration-data ${env.DATA_FILE_2} `
                                    --iteration-data ${env.DATA_FILE_3}
                                
                                # Extraer estadísticas del reporte JSON
                                \$jsonContent = Get-Content "\${jsonReport}" -Raw | ConvertFrom-Json
                                \$total = \$jsonContent.run.stats.iterations.total
                                \$failed = \$jsonContent.run.stats.iterations.failed
                                \$passed = \$total - \$failed
                                
                                Write-Host "Resultados para \${folderName}:"
                                Write-Host "  Total: \${total}"
                                Write-Host "  Exitosos: \${passed}"
                                Write-Host "  Fallidos: \${failed}"
                                
                                # Guardar resultados en archivo
                                "\${folderName},\${total},\${passed},\${failed}" | Out-File -FilePath "${env.EXECUTION_FOLDER}/resultados.csv" -Append -Encoding UTF8
                            """
                        }
                    }
                }
            }
        }
        
        stage('Generar Resumen de Ejecución') {
            steps {
                powershell '''
                    # Crear archivo de resumen
                    $summaryFile = "${env.EXECUTION_FOLDER}/resumen_ejecucion.txt"
                    
                    Write-Host "Generando resumen de ejecución..." | Out-File -FilePath $summaryFile
                    Write-Host "=========================================" | Out-File -FilePath $summaryFile -Append
                    Write-Host "Ejecución: ACHDATA" | Out-File -FilePath $summaryFile -Append
                    Write-Host "Fecha: ${env.EXECUTION_DATE}" | Out-File -FilePath $summaryFile -Append
                    Write-Host "Hora: ${env.EXECUTION_TIME}" | Out-File -FilePath $summaryFile -Append
                    Write-Host "=========================================" | Out-File -FilePath $summaryFile -Append
                    
                    if (Test-Path "${env.EXECUTION_FOLDER}/resultados.csv") {
                        $results = Import-Csv "${env.EXECUTION_FOLDER}/resultados.csv" -Header "Carpeta", "Total", "Exitosos", "Fallidos"
                        
                        foreach ($result in $results) {
                            $line = "$($result.Carpeta.PadRight(10)) | Total: $($result.Total.PadLeft(3)) | Exitosos: $($result.Exitosos.PadLeft(3)) | Fallidos: $($result.Fallidos.PadLeft(3))"
                            Write-Host $line | Out-File -FilePath $summaryFile -Append
                        }
                    }
                    
                    # Contar archivos HTML generados
                    $htmlReports = Get-ChildItem "${env.EXECUTION_FOLDER}" -Filter "*.html"
                    Write-Host "" | Out-File -FilePath $summaryFile -Append
                    Write-Host "Total de reportes HTML generados: $($htmlReports.Count)" | Out-File -FilePath $summaryFile -Append
                '''
            }
        }
        
        stage('Enviar Reportes por Correo') {
            steps {
                script {
                    // Buscar todos los reportes HTML
                    def htmlReports = powershell(returnStdout: true, script: '''
                        Get-ChildItem "${env.EXECUTION_FOLDER}" -Filter "*.html" | 
                        Select-Object -ExpandProperty FullName
                    ''').trim().split('\r\n')
                    
                    // Preparar cuerpo del correo
                    def emailBody = """
                    <html>
                    <body>
                    <h2>Reporte de Ejecución Postman - ACHDATA</h2>
                    <p><strong>Fecha de ejecución:</strong> ${env.EXECUTION_DATE} ${env.EXECUTION_TIME}</p>
                    <br>
                    <h3>Resumen de Resultados:</h3>
                    <table border="1" cellpadding="5" cellspacing="0">
                        <tr style="background-color: #f2f2f2;">
                            <th>Carpeta</th>
                            <th>Total Casos</th>
                            <th>Exitosos</th>
                            <th>Fallidos</th>
                        </tr>
                    """
                    
                    // Leer resultados del archivo CSV
                    if (fileExists("${env.EXECUTION_FOLDER}/resultados.csv")) {
                        def results = readCSV file: "${env.EXECUTION_FOLDER}/resultados.csv", noHeaders: true
                        results.each { row ->
                            def folder = row[0]
                            def total = row[1]
                            def passed = row[2]
                            def failed = row[3]
                            
                            emailBody += """
                            <tr>
                                <td>ACHDATA - ${folder}</td>
                                <td>${total}</td>
                                <td style="color: green;">${passed}</td>
                                <td style="color: ${failed != '0' ? 'red' : 'black'};">${failed}</td>
                            </tr>
                            """
                        }
                    }
                    
                    emailBody += """
                    </table>
                    <br>
                    <p><strong>Total de reportes adjuntos:</strong> ${htmlReports.size()}</p>
                    <p>Los reportes HTML se encuentran adjuntos a este correo.</p>
                    </body>
                    </html>
                    """
                    
                    // Enviar correo con reportes adjuntos
                    emailext(
                        to: "${env.EMAIL_TO}",
                        subject: "Reporte de Ejecución Postman - ACHDATA - ${env.EXECUTION_DATE}",
                        body: emailBody,
                        mimeType: 'text/html',
                        attachmentsPattern: "${env.EXECUTION_FOLDER}/**/*.html",
                        from: "jenkins@cbit-online.com",
                        replyTo: "jenkins@cbit-online.com",
                        smtpServer: "${env.SMTP_SERVER}",
                        smtpPort: "${env.SMTP_PORT}",
                        smtpAuth: true,
                        smtpUsername: "${SMTP_USERNAME}", // Definir en Jenkins Credentials
                        smtpPassword: "${SMTP_PASSWORD}"  // Definir en Jenkins Credentials
                    )
                }
            }
        }
    }
    
    post {
        always {
            echo "Proceso de ejecución completado"
            echo "Reportes guardados en: ${env.EXECUTION_FOLDER}"
            
            // Archivar reportes
            archiveArtifacts artifacts: "${env.EXECUTION_FOLDER}/**/*", allowEmptyArchive: true
        }
        success {
            echo 'Ejecución completada correctamente'
        }
        failure {
            echo 'Fallaron las pruebas Postman'
        }
    }
}