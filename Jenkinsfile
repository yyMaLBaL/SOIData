pipeline {
    agent any
    
    parameters {
        choice(
            name: 'COLLECTION_FOLDER',
            choices: ['ALL', 'AT', 'SS', 'CS', 'CER', 'CERCS'],
            description: 'Selecciona la carpeta a ejecutar'
        )
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
        
        stage('Configurar Ejecución') {
            steps {
                script {
                    // Obtener fecha y hora usando PowerShell
                    def dateTimeOutput = powershell(returnStdout: true, script: '''
                        $date = Get-Date -Format "yyyy-MM-dd"
                        $time = Get-Date -Format "HH-mm-ss"
                        Write-Output "${date}_${time}"
                    ''').trim()
                    
                    def executionFolder = "Ejecuciones\ACHDATA\Fecha_${dateTimeOutput.replace('_', '_Hora_')}"
                    
                    // Crear la carpeta directamente
                    powershell """
                        Write-Host "Creando carpeta de ejecución: ${executionFolder}"
                        New-Item -ItemType Directory -Force -Path "${executionFolder}"
                    """
                    
                    // Guardar en variables para usar después
                    env.EXECUTION_FOLDER = executionFolder
                    env.EXECUTION_DATE = dateTimeOutput.split('_')[0]
                    env.EXECUTION_TIME = dateTimeOutput.split('_')[1]
                    
                    echo "Fecha: ${env.EXECUTION_DATE}"
                    echo "Hora: ${env.EXECUTION_TIME}"
                    echo "Carpeta: ${env.EXECUTION_FOLDER}"
                }
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
                                \$executionFolder = "${env.EXECUTION_FOLDER}"
                                \$reportFile = "\$executionFolder\\report_\${folderName}.html"
                                \$jsonReport = "\$executionFolder\\report_\${folderName}.json"
                                
                                Write-Host "========================================="
                                Write-Host "Ejecutando colección para carpeta: \${folderName}"
                                Write-Host "========================================="
                                
                                # Ejecutar Newman con reporte HTML y JSON
                                newman run "Collection/ACHDATA - YY.postman_collection.json" `
                                    -e "Environment/ACHData QA.postman_environment.json" `
                                    --folder "\${folderName}" `
                                    --insecure `
                                    --reporters cli,html,json `
                                    --reporter-html-export "\${reportFile}" `
                                    --reporter-json-export "\${jsonReport}" `
                                    --iteration-data "File/Incluir_Excluir Personas.csv" `
                                    --iteration-data "File/50 Registros.csv" `
                                    --iteration-data "File/cargue_masivo_usuarios.csv"
                                
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
                                    "\${folderName},\${total},\${passed},\${failed}" | Out-File -FilePath "\$executionFolder\resultados.csv" -Append -Encoding UTF8
                                } else {
                                    Write-Host "ADVERTENCIA: No se generó reporte JSON para \${folderName}"
                                    "\${folderName},0,0,0" | Out-File -FilePath "\$executionFolder\resultados.csv" -Append -Encoding UTF8
                                }
                            """
                        }
                    }
                }
            }
        }
        
        stage('Generar Resumen de Ejecución') {
            steps {
                powershell """
                    \$executionFolder = "${env.EXECUTION_FOLDER}"
                    
                    # Crear archivo de resumen
                    \$summaryFile = "\$executionFolder\\resumen_ejecucion.txt"
                    
                    Write-Host "Generando resumen de ejecución..." | Out-File -FilePath \$summaryFile
                    Write-Host "=========================================" | Out-File -FilePath \$summaryFile -Append
                    Write-Host "Ejecución: ACHDATA" | Out-File -FilePath \$summaryFile -Append
                    Write-Host "Fecha: ${env.EXECUTION_DATE}" | Out-File -FilePath \$summaryFile -Append
                    Write-Host "Hora: ${env.EXECUTION_TIME}" | Out-File -FilePath \$summaryFile -Append
                    Write-Host "=========================================" | Out-File -FilePath \$summaryFile -Append
                    Write-Host "" | Out-File -FilePath \$summaryFile -Append
                    Write-Host "RESULTADOS POR CARPETA:" | Out-File -FilePath \$summaryFile -Append
                    Write-Host "-----------------------" | Out-File -FilePath \$summaryFile -Append
                    
                    if (Test-Path "\$executionFolder\\resultados.csv") {
                        \$lines = Get-Content "\$executionFolder\\resultados.csv"
                        
                        foreach (\$line in \$lines) {
                            \$parts = \$line.Split(',')
                            if (\$parts.Count -eq 4) {
                                \$folder = \$parts[0].Trim()
                                \$total = \$parts[1].Trim()
                                \$passed = \$parts[2].Trim()
                                \$failed = \$parts[3].Trim()
                                
                                \$formattedLine = "ACHDATA - \$folder | Total: \$total | Exitosos: \$passed | Fallidos: \$failed"
                                Write-Host \$formattedLine | Out-File -FilePath \$summaryFile -Append
                            }
                        }
                    } else {
                        Write-Host "No se encontró el archivo de resultados" | Out-File -FilePath \$summaryFile -Append
                    }
                    
                    # Contar archivos HTML generados
                    \$htmlReports = Get-ChildItem "\$executionFolder" -Filter "*.html"
                    Write-Host "" | Out-File -FilePath \$summaryFile -Append
                    Write-Host "=========================================" | Out-File -FilePath \$summaryFile -Append
                    Write-Host "Total de reportes HTML generados: \$(\$htmlReports.Count)" | Out-File -FilePath \$summaryFile -Append
                    
                    # Mostrar resumen en consola
                    Write-Host ""
                    Write-Host "=== RESUMEN DE EJECUCIÓN ==="
                    Get-Content \$summaryFile
                """
            }
        }
        
        stage('Enviar Reportes por Correo') {
            steps {
                script {
                    // Primero verificar que existan reportes
                    def htmlFiles = powershell(returnStdout: true, script: """
                        \$executionFolder = "${env.EXECUTION_FOLDER}"
                        if (Test-Path "\$executionFolder") {
                            \$files = Get-ChildItem "\$executionFolder" -Filter "*.html" | Select-Object -ExpandProperty Name
                            if (\$files) {
                                Write-Output "\$files"
                            } else {
                                Write-Output "NO_FILES"
                            }
                        } else {
                            Write-Output "NO_FOLDER"
                        }
                    """).trim()
                    
                    echo "Archivos HTML encontrados: ${htmlFiles}"
                    
                    // Leer resultados para el cuerpo del correo
                    def summaryContent = ""
                    try {
                        summaryContent = readFile("${env.EXECUTION_FOLDER}/resumen_ejecucion.txt")
                    } catch (Exception e) {
                        summaryContent = "No se pudo leer el resumen de ejecución"
                    }
                    
                    // Preparar cuerpo del correo
                    def emailBody = """
                    <html>
                    <body style="font-family: Arial, sans-serif;">
                        <h2>📊 Reporte de Ejecución Postman - ACHDATA</h2>
                        <p><strong>Fecha de ejecución:</strong> ${env.EXECUTION_DATE} ${env.EXECUTION_TIME}</p>
                        
                        <h3>Resumen de Resultados:</h3>
                        <pre style="background-color: #f5f5f5; padding: 15px; border-radius: 5px;">
                    ${summaryContent}
                        </pre>
                        
                        <p><strong>📎 Reportes adjuntos:</strong> ${htmlFiles.split('\\r\\n').size()} archivos HTML</p>
                        
                        <hr>
                        <p><em>Este correo fue generado automáticamente por Jenkins.</em></p>
                    </body>
                    </html>
                    """
                    
                    // Enviar correo
                    echo "Enviando correo a: yeinerballesta@cbit-online.com"
                    echo "Desde servidor: email.periferia-it.com:587"
                    
                    // Si el plugin de email está configurado
                    try {
                        emailext(
                            to: 'yeinerballesta@cbit-online.com',
                            subject: "Reporte de Ejecución Postman - ACHDATA - ${env.EXECUTION_DATE}",
                            body: emailBody,
                            mimeType: 'text/html',
                            attachmentsPattern: "${env.EXECUTION_FOLDER}/**/*.html",
                            from: 'jenkins@cbit-online.com',
                            smtpServer: 'email.periferia-it.com',
                            smtpPort: '587'
                            // Agregar credenciales si son necesarias
                            // smtpUsername: 'tu_usuario',
                            // smtpPassword: 'tu_password',
                            // smtpAuth: true
                        )
                        echo "Correo enviado exitosamente"
                    } catch (Exception e) {
                        echo "No se pudo enviar el correo: ${e.message}"
                        echo "Cuerpo del correo preparado pero no enviado"
                        echo emailBody
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo "Proceso de ejecución completado"
            echo "Reportes guardados en: ${env.EXECUTION_FOLDER}"
            
            // Archivar reportes
            try {
                archiveArtifacts artifacts: "${env.EXECUTION_FOLDER}/**/*", allowEmptyArchive: true
            } catch (Exception e) {
                echo "No se pudieron archivar los artefactos: ${e.message}"
            }
        }
        success {
            echo 'Ejecución completada correctamente'
        }
        failure {
            echo 'Fallaron las pruebas Postman'
        }
    }
}