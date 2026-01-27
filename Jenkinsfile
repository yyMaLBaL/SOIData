pipeline {
    agent any
    
    parameters {
        choice(
            name: 'COLLECTION_FOLDER',
            choices: ['ALL', 'AT', 'SS', 'CS', 'CER', 'CERCS'],
            description: 'Selecciona la carpeta a ejecutar'
        )
    }
    
    environment {
        COLLECTION_PATH = "Collection/\"ACHDATA - YY.postman_collection.json\""
        ENVIRONMENT_PATH = "Environment/\"ACHData QA.postman_environment.json\""
        DATA_FILE_1 = "File/\"Incluir_Excluir Personas.csv\""
        DATA_FILE_2 = "File/\"50 Registros.csv\""
        DATA_FILE_3 = "File/\"cargue_masivo_usuarios.csv\""
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
        
        stage('Inicializar Variables') {
            steps {
                script {
                    // Obtener fecha y hora en formato Windows
                    def dateTime = powershell(returnStdout: true, script: '''
                        Get-Date -Format "yyyy-MM-dd_HH-mm-ss"
                    ''').trim()
                    
                    def parts = dateTime.split('_')
                    env.EXECUTION_DATE = parts[0]
                    env.EXECUTION_TIME = parts[1]
                    env.EXECUTION_FOLDER = "Ejecuciones\\ACHDATA\\Fecha_${env.EXECUTION_DATE}_Hora_${env.EXECUTION_TIME}"
                    
                    echo "Fecha: ${env.EXECUTION_DATE}"
                    echo "Hora: ${env.EXECUTION_TIME}"
                    echo "Carpeta: ${env.EXECUTION_FOLDER}"
                }
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
                    
                    folders.each { folder ->
                        stage("Ejecutar ${folder}") {
                            powershell """
                                \$folderName = "${folder}"
                                \$reportFile = "\${env:EXECUTION_FOLDER}\\report_\${folderName}.html"
                                \$jsonReport = "\${env:EXECUTION_FOLDER}\\report_\${folderName}.json"
                                
                                Write-Host "========================================="
                                Write-Host "Ejecutando colección para carpeta: \${folderName}"
                                Write-Host "========================================="
                                
                                # Ejecutar Newman con reporte HTML y JSON
                                newman run "${env.COLLECTION_PATH}" `
                                    -e "${env.ENVIRONMENT_PATH}" `
                                    --folder "\${folderName}" `
                                    --insecure `
                                    --reporters cli,html,json `
                                    --reporter-html-export "\${reportFile}" `
                                    --reporter-json-export "\${jsonReport}" `
                                    --iteration-data "${env.DATA_FILE_1}" `
                                    --iteration-data "${env.DATA_FILE_2}" `
                                    --iteration-data "${env.DATA_FILE_3}"
                                
                                # Verificar si se generó el reporte JSON
                                if (Test-Path "\${jsonReport}") {
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
                                    "\${folderName},\${total},\${passed},\${failed}" | Out-File -FilePath "\${env:EXECUTION_FOLDER}\\resultados.csv" -Append -Encoding UTF8
                                } else {
                                    Write-Host "ADVERTENCIA: No se generó reporte JSON para \${folderName}"
                                }
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
                    $summaryFile = "${env:EXECUTION_FOLDER}\\resumen_ejecucion.txt"
                    
                    Write-Host "Generando resumen de ejecución..." | Out-File -FilePath $summaryFile
                    Write-Host "=========================================" | Out-File -FilePath $summaryFile -Append
                    Write-Host "Ejecución: ACHDATA" | Out-File -FilePath $summaryFile -Append
                    Write-Host "Fecha: ${env:EXECUTION_DATE}" | Out-File -FilePath $summaryFile -Append
                    Write-Host "Hora: ${env:EXECUTION_TIME}" | Out-File -FilePath $summaryFile -Append
                    Write-Host "=========================================" | Out-File -FilePath $summaryFile -Append
                    Write-Host "" | Out-File -FilePath $summaryFile -Append
                    Write-Host "RESULTADOS POR CARPETA:" | Out-File -FilePath $summaryFile -Append
                    Write-Host "-----------------------" | Out-File -FilePath $summaryFile -Append
                    
                    if (Test-Path "${env:EXECUTION_FOLDER}\\resultados.csv") {
                        $lines = Get-Content "${env:EXECUTION_FOLDER}\\resultados.csv"
                        
                        foreach ($line in $lines) {
                            $parts = $line.Split(',')
                            if ($parts.Count -eq 4) {
                                $folder = $parts[0].Trim()
                                $total = $parts[1].Trim()
                                $passed = $parts[2].Trim()
                                $failed = $parts[3].Trim()
                                
                                $formattedLine = "ACHDATA - $folder | Total: $total | Exitosos: $passed | Fallidos: $failed"
                                Write-Host $formattedLine | Out-File -FilePath $summaryFile -Append
                            }
                        }
                    } else {
                        Write-Host "No se encontró el archivo de resultados" | Out-File -FilePath $summaryFile -Append
                    }
                    
                    # Contar archivos HTML generados
                    $htmlReports = Get-ChildItem "${env:EXECUTION_FOLDER}" -Filter "*.html"
                    Write-Host "" | Out-File -FilePath $summaryFile -Append
                    Write-Host "=========================================" | Out-File -FilePath $summaryFile -Append
                    Write-Host "Total de reportes HTML generados: $($htmlReports.Count)" | Out-File -FilePath $summaryFile -Append
                    
                    # Mostrar resumen en consola
                    Write-Host ""
                    Write-Host "=== RESUMEN DE EJECUCIÓN ==="
                    Get-Content $summaryFile
                '''
            }
        }
        
        stage('Enviar Reportes por Correo') {
            steps {
                script {
                    // Buscar todos los reportes HTML
                    def htmlFiles = powershell(returnStdout: true, script: '''
                        Get-ChildItem "${env:EXECUTION_FOLDER}" -Filter "*.html" | 
                        Select-Object -ExpandProperty Name
                    ''').trim()
                    
                    def htmlFileList = htmlFiles ? htmlFiles.split('\\r\\n') : []
                    
                    // Leer el resumen para el cuerpo del correo
                    def summaryContent = ""
                    if (fileExists("${env.EXECUTION_FOLDER}/resumen_ejecucion.txt")) {
                        summaryContent = readFile("${env.EXECUTION_FOLDER}/resumen_ejecucion.txt")
                    }
                    
                    // Preparar cuerpo del correo HTML
                    def emailBody = """
                    <html>
                    <head>
                        <style>
                            body { font-family: Arial, sans-serif; }
                            table { border-collapse: collapse; width: 100%; }
                            th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
                            th { background-color: #f2f2f2; }
                            .success { color: green; }
                            .failure { color: red; }
                            .info { background-color: #e7f3fe; padding: 15px; border-left: 6px solid #2196F3; }
                        </style>
                    </head>
                    <body>
                        <div class="info">
                            <h2>📊 Reporte de Ejecución Postman - ACHDATA</h2>
                            <p><strong>Fecha de ejecución:</strong> ${env.EXECUTION_DATE} ${env.EXECUTION_TIME}</p>
                        </div>
                        
                        <br>
                        <h3>📋 Resumen de Resultados:</h3>
                        <table>
                            <tr>
                                <th>Carpeta</th>
                                <th>Total Casos</th>
                                <th>Exitosos</th>
                                <th>Fallidos</th>
                            </tr>
                    """
                    
                    // Leer resultados del archivo CSV
                    if (fileExists("${env.EXECUTION_FOLDER}/resultados.csv")) {
                        def results = readFile("${env.EXECUTION_FOLDER}/resultados.csv")
                        def lines = results.split('\\r\\n')
                        
                        lines.each { line ->
                            def parts = line.split(',')
                            if (parts.size() == 4) {
                                def folder = parts[0].trim()
                                def total = parts[1].trim()
                                def passed = parts[2].trim()
                                def failed = parts[3].trim()
                                
                                emailBody += """
                                <tr>
                                    <td><strong>ACHDATA - ${folder}</strong></td>
                                    <td>${total}</td>
                                    <td class="success">${passed}</td>
                                    <td ${failed != '0' ? 'class="failure"' : ''}>${failed}</td>
                                </tr>
                                """
                            }
                        }
                    }
                    
                    emailBody += """
                        </table>
                        <br>
                        <div>
                            <h3>📎 Archivos Adjuntos:</h3>
                            <p>Se adjuntan ${htmlFileList.size()} reportes HTML:</p>
                            <ul>
                    """
                    
                    htmlFileList.each { fileName ->
                        emailBody += "<li>${fileName}</li>"
                    }
                    
                    emailBody += """
                            </ul>
                        </div>
                        <br>
                        <div style="margin-top: 20px; padding: 10px; background-color: #f9f9f9;">
                            <p><strong>Nota:</strong> Este correo fue generado automáticamente por Jenkins.</p>
                        </div>
                    </body>
                    </html>
                    """
                    
                    // Enviar correo (versión simplificada primero)
                    echo "Preparando envío de correo a: ${env.EMAIL_TO}"
                    echo "Adjuntando ${htmlFileList.size()} archivos HTML"
                    echo "Carpeta de reportes: ${env.EXECUTION_FOLDER}"
                    
                    // Si tienes configurado el plugin de email, usa esto:
                    emailext(
                        to: "${env.EMAIL_TO}",
                        subject: "Reporte de Ejecución Postman - ACHDATA - ${env.EXECUTION_DATE}",
                        body: emailBody,
                        mimeType: 'text/html',
                        attachmentsPattern: "${env.EXECUTION_FOLDER}/**/*.html",
                        from: "jenkins@cbit-online.com",
                        smtpServer: "${env.SMTP_SERVER}",
                        smtpPort: "${env.SMTP_PORT}"
                        // Agrega credenciales si son necesarias
                        // smtpUsername: credentials('smtp-user').username,
                        // smtpPassword: credentials('smtp-pass').password
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
            
            // Limpiar archivos temporales si es necesario
            cleanWs()
        }
        success {
            echo 'Ejecución completada correctamente'
        }
        failure {
            echo 'Fallaron las pruebas Postman'
        }
    }
}