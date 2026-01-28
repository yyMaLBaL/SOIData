pipeline {
    agent any
    
    parameters {
        choice(
            name: 'AMBIENTE',
            choices: ['ALL', 'AT', 'SS', 'CS', 'CER', 'CERCS'],
            description: 'Selecciona el ambiente a ejecutar'
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
        
        stage('Configurar Ejecucion') {
            steps {
                script {
                    def dateTimeOutput = powershell(returnStdout: true, script: '''
                        $date = Get-Date -Format "yyyy-MM-dd"
                        $time = Get-Date -Format "HH-mm-ss"
                        Write-Output "${date}_${time}"
                    ''').trim()
                    
                    // Crear carpeta Executions en la misma ruta de los archivos de Postman
                    def basePath = "Ejecuciones/ACHDATA"
                    def executionFolder = "${basePath}/Fecha_${dateTimeOutput.split('_')[0]}_Hora_${dateTimeOutput.split('_')[1]}"
                    
                    powershell """
                        Write-Host "Creando estructura de carpetas..."
                        
                        # Crear carpeta base si no existe
                        if (!(Test-Path "${basePath}")) {
                            New-Item -ItemType Directory -Force -Path "${basePath}"
                            Write-Host "Carpeta base creada: ${basePath}"
                        }
                        
                        # Crear carpeta de ejecucion especifica
                        New-Item -ItemType Directory -Force -Path "${executionFolder}"
                        Write-Host "Carpeta de ejecucion creada: ${executionFolder}"
                        
                        # Crear subcarpetas para organizacion
                        New-Item -ItemType Directory -Force -Path "${executionFolder}/Reportes_HTML"
                        New-Item -ItemType Directory -Force -Path "${executionFolder}/Reportes_JSON"
                        Write-Host "Subcarpetas creadas"
                    """
                    
                    env.EXECUTION_FOLDER = executionFolder
                    env.EXECUTION_BASE_PATH = basePath
                    env.EXECUTION_DATE = dateTimeOutput.split('_')[0]
                    env.EXECUTION_TIME = dateTimeOutput.split('_')[1]
                    
                    echo "Fecha: ${env.EXECUTION_DATE}"
                    echo "Hora: ${env.EXECUTION_TIME}"
                    echo "Carpeta base: ${env.EXECUTION_BASE_PATH}"
                    echo "Carpeta ejecucion: ${env.EXECUTION_FOLDER}"
                }
            }
        }
        
        stage('Ejecutar Coleccion Postman') {
            steps {
                script {
                    // Definir ambientes a ejecutar
                    def ambientes = params.AMBIENTE == 'ALL' ? 
                        ['AT', 'SS', 'CS', 'CER', 'CERCS'] : 
                        [params.AMBIENTE]
                    
                    // Lista de carpetas base (sin prefijo de ambiente)
                    def carpetasBase = [
                        'Autenticacion',
                        'Detallada Natural Y Juridica'
                        // Agrega aquí el resto de carpetas que veas en tu colección
                    ]
                    
                    ambientes.each { ambiente ->
                        carpetasBase.each { carpetaBase ->
                            def folderName = "${ambiente} - ${carpetaBase}"
                            
                            stage("${ambiente} - ${carpetaBase}") {
                                def exitCode = powershell(returnStatus: true, script: """
                                    \$folderName = "${folderName}"
                                    \$ambiente = "${ambiente}"
                                    \$carpetaBase = "${carpetaBase}"
                                    \$executionFolder = "${env.EXECUTION_FOLDER}".Replace('/', '\\')
                                    \$safeFileName = "\${ambiente}_\${carpetaBase}".Replace(' ', '_').Replace('-', '_')
                                    \$reportFile = "\$executionFolder\\Reportes_HTML\\report_\${safeFileName}.html"
                                    \$jsonReport = "\$executionFolder\\Reportes_JSON\\report_\${safeFileName}.json"
                                    \$resultadosFile = "\$executionFolder\\resultados.csv"
                                    
                                    Write-Host "========================================="
                                    Write-Host "Intentando ejecutar: \${folderName}"
                                    Write-Host "========================================="
                                    
                                    \$newmanExitCode = 0
                                    \$folderExists = \$false
                                    \$outputText = ""
                                    
                                    try {
                                        # Verificar que los archivos de entrada existen
                                        if (!(Test-Path "Collection/ACHDATA - YY.postman_collection.json")) {
                                            Write-Host "[ERROR] No se encuentra la colección"
                                            exit 1
                                        }
                                        
                                        if (!(Test-Path "Environment/ACHData QA.postman_environment.json")) {
                                            Write-Host "[ERROR] No se encuentra el entorno"
                                            exit 1
                                        }
                                        
                                        \$outputText = newman run "Collection/ACHDATA - YY.postman_collection.json" `
                                            -e "Environment/ACHData QA.postman_environment.json" `
                                            --folder "\${folderName}" `
                                            --insecure `
                                            --reporters cli,html,json `
                                            --reporter-html-export "\${reportFile}" `
                                            --reporter-json-export "\${jsonReport}" 2>&1 | Out-String
                                        
                                        \$newmanExitCode = \$LASTEXITCODE
                                        Write-Host \$outputText
                                        
                                        if (\$outputText -match "Unable to find a folder") {
                                            Write-Host "[INFO] Carpeta '\${folderName}' no encontrada"
                                            \$folderExists = \$false
                                        } else {
                                            \$folderExists = \$true
                                        }
                                        
                                    } catch {
                                        Write-Host "Error: \$_"
                                        \$newmanExitCode = 1
                                        \$folderExists = \$false
                                    }
                                    
                                    if (\$folderExists -and (Test-Path "\${jsonReport}")) {
                                        try {
                                            \$jsonContent = Get-Content "\${jsonReport}" -Raw | ConvertFrom-Json
                                            \$totalRequests = \$jsonContent.run.stats.requests.total
                                            \$failedRequests = \$jsonContent.run.stats.requests.failed
                                            \$passedRequests = \$totalRequests - \$failedRequests
                                            
                                            \$totalAssertions = \$jsonContent.run.stats.assertions.total
                                            \$failedAssertions = \$jsonContent.run.stats.assertions.failed
                                            \$passedAssertions = \$totalAssertions - \$failedAssertions
                                            
                                            Write-Host ""
                                            Write-Host "[OK] Resultados:"
                                            Write-Host "  Requests: \${totalRequests} (OK:\${passedRequests} / FAIL:\${failedRequests})"
                                            Write-Host "  Assertions: \${totalAssertions} (OK:\${passedAssertions} / FAIL:\${failedAssertions})"
                                            Write-Host ""
                                            
                                            if (!(Test-Path "\${resultadosFile}")) {
                                                "AMBIENTE,CARPETA,ESTADO,TOTAL_REQUESTS,REQUESTS_OK,REQUESTS_FAIL,TOTAL_ASSERTIONS,ASSERTIONS_OK,ASSERTIONS_FAIL" | Out-File -FilePath "\${resultadosFile}" -Encoding UTF8
                                            }
                                            
                                            "\${ambiente},\${carpetaBase},EJECUTADO,\${totalRequests},\${passedRequests},\${failedRequests},\${totalAssertions},\${passedAssertions},\${failedAssertions}" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                        } catch {
                                            Write-Host "Error procesando JSON: \$_"
                                            if (!(Test-Path "\${resultadosFile}")) {
                                                "AMBIENTE,CARPETA,ESTADO,TOTAL_REQUESTS,REQUESTS_OK,REQUESTS_FAIL,TOTAL_ASSERTIONS,ASSERTIONS_OK,ASSERTIONS_FAIL" | Out-File -FilePath "\${resultadosFile}" -Encoding UTF8
                                            }
                                            "\${ambiente},\${carpetaBase},ERROR,0,0,0,0,0,0" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                        }
                                    } else {
                                        Write-Host "[INFO] Carpeta no disponible"
                                        if (!(Test-Path "\${resultadosFile}")) {
                                            "AMBIENTE,CARPETA,ESTADO,TOTAL_REQUESTS,REQUESTS_OK,REQUESTS_FAIL,TOTAL_ASSERTIONS,ASSERTIONS_OK,ASSERTIONS_FAIL" | Out-File -FilePath "\${resultadosFile}" -Encoding UTF8
                                        }
                                        "\${ambiente},\${carpetaBase},NO_DISPONIBLE,0,0,0,0,0,0" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                    }
                                    
                                    exit 0
                                """)
                                
                                if (exitCode != 0) {
                                    echo "[WARN] Hubo un problema, pero continuamos"
                                }
                            }
                        }
                    }
                }
            }
        }
        
        stage('Generar Resumen') {
            steps {
                powershell """
                    \$executionFolder = "${env.EXECUTION_FOLDER}".Replace('/', '\\')
                    \$summaryFile = "\$executionFolder\\resumen_ejecucion.txt"
                    \$resultadosFile = "\$executionFolder\\resultados.csv"
                    
                    "=========================================" | Out-File -FilePath \$summaryFile
                    "RESUMEN DE EJECUCION - ACHDATA" | Out-File -FilePath \$summaryFile -Append
                    "=========================================" | Out-File -FilePath \$summaryFile -Append
                    "Fecha: ${env.EXECUTION_DATE}" | Out-File -FilePath \$summaryFile -Append
                    "Hora: ${env.EXECUTION_TIME}" | Out-File -FilePath \$summaryFile -Append
                    "Ambiente: ${params.AMBIENTE}" | Out-File -FilePath \$summaryFile -Append
                    "Carpeta de ejecucion: \$executionFolder" | Out-File -FilePath \$summaryFile -Append
                    "=========================================" | Out-File -FilePath \$summaryFile -Append
                    "" | Out-File -FilePath \$summaryFile -Append
                    
                    if (Test-Path "\$resultadosFile") {
                        "RESULTADOS:" | Out-File -FilePath \$summaryFile -Append
                        "-----------------------" | Out-File -FilePath \$summaryFile -Append
                        "" | Out-File -FilePath \$summaryFile -Append
                        
                        \$lines = Get-Content "\$resultadosFile"
                        \$ejecutados = 0
                        \$noDisponibles = 0
                        \$errores = 0
                        \$totalRequests = 0
                        \$totalPassedRequests = 0
                        \$totalFailedRequests = 0
                        
                        foreach (\$line in \$lines) {
                            # Saltar encabezado
                            if (\$line -match "^AMBIENTE,") {
                                continue
                            }
                            
                            \$parts = \$line.Split(',')
                            if (\$parts.Count -eq 9) {
                                \$amb = \$parts[0].Trim()
                                \$carpeta = \$parts[1].Trim()
                                \$estado = \$parts[2].Trim()
                                \$reqTotal = [int]\$parts[3].Trim()
                                \$reqPassed = [int]\$parts[4].Trim()
                                \$reqFailed = [int]\$parts[5].Trim()
                                
                                if (\$estado -eq "EJECUTADO") {
                                    \$ejecutados++
                                    \$totalRequests += \$reqTotal
                                    \$totalPassedRequests += \$reqPassed
                                    \$totalFailedRequests += \$reqFailed
                                    
                                    if (\$reqFailed -eq 0) {
                                        "[OK] \$amb - \$carpeta" | Out-File -FilePath \$summaryFile -Append
                                    } else {
                                        "[WARN] \$amb - \$carpeta" | Out-File -FilePath \$summaryFile -Append
                                    }
                                    "     Requests: \$reqTotal (OK:\$reqPassed / FAIL:\$reqFailed)" | Out-File -FilePath \$summaryFile -Append
                                } elseif (\$estado -eq "NO_DISPONIBLE") {
                                    \$noDisponibles++
                                    "[N/A] \$amb - \$carpeta" | Out-File -FilePath \$summaryFile -Append
                                } else {
                                    \$errores++
                                    "[ERROR] \$amb - \$carpeta" | Out-File -FilePath \$summaryFile -Append
                                }
                                "" | Out-File -FilePath \$summaryFile -Append
                            }
                        }
                        
                        "=========================================" | Out-File -FilePath \$summaryFile -Append
                        "RESUMEN:" | Out-File -FilePath \$summaryFile -Append
                        "-----------------------" | Out-File -FilePath \$summaryFile -Append
                        "Carpetas ejecutadas: \$ejecutados" | Out-File -FilePath \$summaryFile -Append
                        "Carpetas no disponibles: \$noDisponibles" | Out-File -FilePath \$summaryFile -Append
                        "Carpetas con error: \$errores" | Out-File -FilePath \$summaryFile -Append
                        "" | Out-File -FilePath \$summaryFile -Append
                        
                        if (\$ejecutados -gt 0) {
                            "TOTALES (ejecutadas):" | Out-File -FilePath \$summaryFile -Append
                            "Requests totales: \$totalRequests" | Out-File -FilePath \$summaryFile -Append
                            "Requests exitosos: \$totalPassedRequests" | Out-File -FilePath \$summaryFile -Append
                            "Requests fallidos: \$totalFailedRequests" | Out-File -FilePath \$summaryFile -Append
                            
                            if (\$totalRequests -gt 0) {
                                \$successRate = [math]::Round((\$totalPassedRequests / \$totalRequests) * 100, 2)
                                "Tasa de exito: \$successRate%" | Out-File -FilePath \$summaryFile -Append
                            }
                        }
                    } else {
                        "No se encontraron resultados" | Out-File -FilePath \$summaryFile -Append
                    }
                    
                    \$htmlReports = Get-ChildItem "\$executionFolder\\Reportes_HTML" -Filter "*.html" -ErrorAction SilentlyContinue
                    \$jsonReports = Get-ChildItem "\$executionFolder\\Reportes_JSON" -Filter "*.json" -ErrorAction SilentlyContinue
                    
                    "" | Out-File -FilePath \$summaryFile -Append
                    "=========================================" | Out-File -FilePath \$summaryFile -Append
                    "Archivos generados:" | Out-File -FilePath \$summaryFile -Append
                    "  Reportes HTML: \$(\$htmlReports.Count)" | Out-File -FilePath \$summaryFile -Append
                    "  Reportes JSON: \$(\$jsonReports.Count)" | Out-File -FilePath \$summaryFile -Append
                    "  Archivo CSV: \$(if (Test-Path "\$resultadosFile") { 'SI' } else { 'NO' })" | Out-File -FilePath \$summaryFile -Append
                    
                    Write-Host ""
                    Write-Host "=== RESUMEN ==="
                    Get-Content \$summaryFile
                """
            }
        }
        
        stage('Enviar Reportes por Correo') {
            steps {
                script {
                    def htmlFiles = powershell(returnStdout: true, script: """
                        \$executionFolder = "${env.EXECUTION_FOLDER}".Replace('/', '\\')
                        \$htmlFolder = "\$executionFolder\\Reportes_HTML"
                        if (Test-Path \$htmlFolder) {
                            \$files = Get-ChildItem \$htmlFolder -Filter "*.html" -ErrorAction SilentlyContinue
                            if (\$files) {
                                Write-Output (\$files.Name -join ",")
                            } else {
                                Write-Output "NO_FILES"
                            }
                        } else {
                            Write-Output "NO_FOLDER"
                        }
                    """).trim()
                    
                    echo "Archivos HTML encontrados: ${htmlFiles}"
                    
                    def summaryContent = ""
                    try {
                        summaryContent = readFile("${env.EXECUTION_FOLDER}/resumen_ejecucion.txt")
                    } catch (Exception e) {
                        summaryContent = "No se pudo leer el resumen: ${e.message}"
                    }
                    
                    def emailBody = """
                    <html>
                    <body style="font-family: Arial, sans-serif;">
                        <h2>Reporte de Ejecucion Postman - ACHDATA</h2>
                        <p><strong>Fecha:</strong> ${env.EXECUTION_DATE} a las ${env.EXECUTION_TIME}</p>
                        <p><strong>Ambiente:</strong> ${params.AMBIENTE}</p>
                        <p><strong>Carpeta de ejecucion:</strong> ${env.EXECUTION_FOLDER}</p>
                        
                        <h3>Resumen:</h3>
                        <pre style="background-color: #f5f5f5; padding: 15px; border-radius: 5px; font-size: 12px;">
${summaryContent}
                        </pre>
                        
                        <p><strong>Reportes adjuntos:</strong> ${htmlFiles != 'NO_FILES' && htmlFiles != 'NO_FOLDER' ? htmlFiles.split(',').size() : 0} archivos HTML</p>
                        
                        <hr>
                        <p style="font-size: 11px; color: #666;"><em>Generado automaticamente por Jenkins</em></p>
                    </body>
                    </html>
                    """
                    
                    echo "Enviando correo a: yeinerballesta@cbit-online.com"
                    
                    try {
                        emailext(
                            to: 'yeinerballesta@cbit-online.com',
                            subject: "Reporte Postman ACHDATA - ${params.AMBIENTE} - ${env.EXECUTION_DATE}",
                            body: emailBody,
                            mimeType: 'text/html',
                            attachmentsPattern: "${env.EXECUTION_FOLDER}/**/*",
                            from: 'jenkins@cbit-online.com'
                        )
                        echo "[OK] Correo enviado"
                    } catch (Exception e) {
                        echo "[WARN] No se pudo enviar correo: ${e.message}"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "Proceso completado"
                echo "Reportes en: ${env.EXECUTION_FOLDER}"
                echo "Ruta base: ${env.EXECUTION_BASE_PATH}"
                
                try {
                    archiveArtifacts artifacts: "${env.EXECUTION_FOLDER}/**/*", allowEmptyArchive: true
                    echo "[OK] Artefactos archivados"
                } catch (Exception e) {
                    echo "[WARN] No se archivaron: ${e.message}"
                }
            }
        }
        success {
            echo '[OK] Pipeline completado'
        }
        failure {
            echo '[ERROR] Pipeline fallo'
        }
    }
}