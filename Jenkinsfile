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
        
        stage('Verificar Archivos') {
            steps {
                powershell '''
                    Write-Host "=== VERIFICANDO ARCHIVOS ==="
                    Write-Host "Directorio actual: $(Get-Location)"
                    Write-Host ""
                    
                    # Verificar colección
                    $collectionPath = "Collection/ACHDATA - YY.postman_collection.json"
                    if (Test-Path $collectionPath) {
                        Write-Host "[OK] Colección encontrada: $collectionPath"
                        Write-Host "Tamaño: $((Get-Item $collectionPath).Length) bytes"
                    } else {
                        Write-Host "[ERROR] Colección NO encontrada"
                    }
                    
                    Write-Host ""
                    
                    # Verificar entorno
                    $envPath = "Environment/ACHData QA.postman_environment.json"
                    if (Test-Path $envPath) {
                        Write-Host "[OK] Entorno encontrado: $envPath"
                    } else {
                        Write-Host "[ERROR] Entorno NO encontrado"
                    }
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
                    
                    def executionFolder = "Executions\\ACHDATA\\Fecha_${dateTimeOutput.replace('_', '_Hora_')}"
                    
                    powershell """
                        Write-Host "Creando carpeta de ejecucion: ${executionFolder}"
                        New-Item -ItemType Directory -Force -Path "${executionFolder}"
                    """
                    
                    env.EXECUTION_FOLDER = executionFolder
                    env.EXECUTION_DATE = dateTimeOutput.split('_')[0]
                    env.EXECUTION_TIME = dateTimeOutput.split('_')[1]
                    
                    echo "Fecha: ${env.EXECUTION_DATE}"
                    echo "Hora: ${env.EXECUTION_TIME}"
                    echo "Carpeta: ${env.EXECUTION_FOLDER}"
                }
            }
        }
        
        stage('Listar Carpetas de Coleccion') {
            steps {
                script {
                    // Script simple para listar carpetas
                    def folderList = powershell(returnStdout: true, script: '''
                        $collectionPath = "Collection/ACHDATA - YY.postman_collection.json"
                        
                        Write-Host "=== BUSCANDO CARPETAS EN LA COLECCIÓN ==="
                        
                        if (Test-Path $collectionPath) {
                            # Intentar leer el archivo como texto y buscar nombres
                            $content = Get-Content $collectionPath -Raw
                            
                            # Patrón simple para buscar nombres
                            $pattern = '"name"\s*:\s*"([^"]+)"'
                            $matches = [regex]::Matches($content, $pattern)
                            
                            Write-Host "Se encontraron $($matches.Count) nombres:"
                            Write-Host "----------------------------------------"
                            
                            $count = 0
                            foreach ($match in $matches) {
                                $name = $match.Groups[1].Value
                                $count++
                                Write-Host "$count. $name"
                            }
                            
                            Write-Host ""
                            Write-Host "=== MOSTRANDO PRIMEROS 20 NOMBRES ÚNICOS ==="
                            
                            $uniqueNames = @{}
                            foreach ($match in $matches) {
                                $name = $match.Groups[1].Value
                                if (-not $uniqueNames.ContainsKey($name)) {
                                    $uniqueNames[$name] = $true
                                }
                            }
                            
                            $uniqueList = $uniqueNames.Keys | Sort-Object
                            $i = 0
                            foreach ($name in $uniqueList) {
                                $i++
                                Write-Host "$i. $name"
                                if ($i -ge 20) {
                                    Write-Host "... (más nombres omitidos)"
                                    break
                                }
                            }
                            
                        } else {
                            Write-Host "ERROR: Archivo no encontrado"
                        }
                    ''')
                    
                    echo "Lista de carpetas:\n${folderList}"
                }
            }
        }
        
        stage('Probar Ejecucion de Carpetas') {
            steps {
                script {
                    // Lista de posibles nombres de carpetas para probar
                    def testFolders = [
                        "Autenticación",
                        "Detallada Natural Y Juridica", 
                        "AT",
                        "SS", 
                        "CS",
                        "CER",
                        "CERCS",
                        "Auth",
                        "Authentication",
                        "Login"
                    ]
                    
                    testFolders.each { folderName ->
                        stage("Probar: ${folderName}") {
                            def result = powershell(returnStdout: true, script: """
                                `$folderName = '${folderName}'
                                `$executionFolder = '${env.EXECUTION_FOLDER}'
                                
                                Write-Host "=== PROBANDO CARPETA: `$folderName ==="
                                
                                # Intentar ejecutar con esta carpeta
                                `$output = newman run "Collection/ACHDATA - YY.postman_collection.json" `
                                    -e "Environment/ACHData QA.postman_environment.json" `
                                    --folder "`$folderName" `
                                    --insecure `
                                    --reporters cli `
                                    --reporter-cli-no-summary 2>&1
                                
                                Write-Host "Salida de Newman:"
                                Write-Host `$output
                                
                                # Verificar resultado
                                if (`$output -match "Unable to find a folder") {
                                    Write-Host "RESULTADO: Carpeta '`$folderName' NO encontrada"
                                    Write-Output "NOT_FOUND"
                                } elseif (`$output -match "iteration") {
                                    Write-Host "RESULTADO: Carpeta '`$folderName' EJECUTADA con éxito"
                                    Write-Output "SUCCESS"
                                } else {
                                    Write-Host "RESULTADO: Resultado inesperado para '`$folderName'"
                                    Write-Output "OTHER"
                                }
                            """).trim()
                            
                            echo "Resultado para '${folderName}': ${result}"
                        }
                    }
                }
            }
        }
        
        stage('Ejecutar Coleccion Completa') {
            steps {
                powershell """
                    `$executionFolder = '${env.EXECUTION_FOLDER}'
                    `$reportFile = "`$executionFolder\\report_completo.html"
                    `$jsonReport = "`$executionFolder\\report_completo.json"
                    `$resultadosFile = "`$executionFolder\\resultados.csv"
                    
                    Write-Host "=== EJECUTANDO COLECCIÓN COMPLETA ==="
                    Write-Host "Esto ejecutará TODOS los requests de la colección"
                    Write-Host "Reportes se guardarán en: `$executionFolder"
                    Write-Host ""
                    
                    # Ejecutar toda la colección
                    newman run "Collection/ACHDATA - YY.postman_collection.json" `
                        -e "Environment/ACHData QA.postman_environment.json" `
                        --insecure `
                        --reporters cli,html,json `
                        --reporter-html-export "`$reportFile" `
                        --reporter-json-export "`$jsonReport"
                    
                    `$exitCode = `$LASTEXITCODE
                    
                    if (Test-Path "`$jsonReport") {
                        try {
                            `$jsonContent = Get-Content "`$jsonReport" -Raw | ConvertFrom-Json
                            `$totalRequests = `$jsonContent.run.stats.requests.total
                            `$failedRequests = `$jsonContent.run.stats.requests.failed
                            `$passedRequests = `$totalRequests - `$failedRequests
                            
                            Write-Host ""
                            Write-Host "=== RESULTADOS ==="
                            Write-Host "Total requests: `$totalRequests"
                            Write-Host "Passed: `$passedRequests"
                            Write-Host "Failed: `$failedRequests"
                            
                            "COLECCION_COMPLETA,EJECUTADO,`$totalRequests,`$passedRequests,`$failedRequests" | Out-File -FilePath "`$resultadosFile" -Encoding UTF8
                            
                        } catch {
                            Write-Host "Error procesando JSON: `$_"
                        }
                    }
                    
                    if (`$exitCode -eq 0) {
                        Write-Host "[OK] Ejecución completada exitosamente"
                    } else {
                        Write-Host "[WARN] Ejecución completada con errores (código: `$exitCode)"
                    }
                """
            }
        }
        
        stage('Generar Resumen') {
            steps {
                powershell """
                    `$executionFolder = '${env.EXECUTION_FOLDER}'
                    `$summaryFile = "`$executionFolder\\resumen_ejecucion.txt"
                    
                    "=========================================" | Out-File -FilePath `$summaryFile
                    "RESUMEN DE EJECUCION - ACHDATA" | Out-File -FilePath `$summaryFile -Append
                    "=========================================" | Out-File -FilePath `$summaryFile -Append
                    "Fecha: ${env.EXECUTION_DATE}" | Out-File -FilePath `$summaryFile -Append
                    "Hora: ${env.EXECUTION_TIME}" | Out-File -FilePath `$summaryFile -Append
                    "Ambiente seleccionado: ${params.AMBIENTE}" | Out-File -FilePath `$summaryFile -Append
                    "=========================================" | Out-File -FilePath `$summaryFile -Append
                    "" | Out-File -FilePath `$summaryFile -Append
                    
                    # Listar reportes generados
                    `$htmlReports = Get-ChildItem "`$executionFolder" -Filter "*.html" -ErrorAction SilentlyContinue
                    if (`$htmlReports) {
                        "REPORTES GENERADOS:" | Out-File -FilePath `$summaryFile -Append
                        "-------------------" | Out-File -FilePath `$summaryFile -Append
                        foreach (`$report in `$htmlReports) {
                            "  - `$(`$report.Name)" | Out-File -FilePath `$summaryFile -Append
                        }
                        "" | Out-File -FilePath `$summaryFile -Append
                    } else {
                        "No se generaron reportes HTML" | Out-File -FilePath `$summaryFile -Append
                        "" | Out-File -FilePath `$summaryFile -Append
                    }
                    
                    # Mostrar resultados si existen
                    `$resultadosFile = "`$executionFolder\\resultados.csv"
                    if (Test-Path `$resultadosFile) {
                        "RESULTADOS:" | Out-File -FilePath `$summaryFile -Append
                        "----------" | Out-File -FilePath `$summaryFile -Append
                        Get-Content `$resultadosFile | Out-File -FilePath `$summaryFile -Append
                        "" | Out-File -FilePath `$summaryFile -Append
                    }
                    
                    Write-Host ""
                    Write-Host "=== RESUMEN FINAL ==="
                    Get-Content `$summaryFile
                """
            }
        }
        
        stage('Enviar Reportes por Correo') {
            steps {
                script {
                    // Verificar si hay reportes
                    def hasReports = powershell(returnStdout: true, script: """
                        `$executionFolder = '${env.EXECUTION_FOLDER}'
                        `$htmlFiles = Get-ChildItem "`$executionFolder" -Filter "*.html" -ErrorAction SilentlyContinue
                        if (`$htmlFiles) {
                            Write-Output "YES"
                        } else {
                            Write-Output "NO"
                        }
                    """).trim()
                    
                    def summaryContent = ""
                    try {
                        summaryContent = readFile("${env.EXECUTION_FOLDER}\\resumen_ejecucion.txt")
                    } catch (Exception e) {
                        summaryContent = "No se pudo leer el resumen: ${e.message}"
                    }
                    
                    def emailBody = """
                    <html>
                    <body style="font-family: Arial, sans-serif;">
                        <h2>Reporte de Ejecución Postman - ACHDATA</h2>
                        <p><strong>Fecha:</strong> ${env.EXECUTION_DATE}</p>
                        <p><strong>Hora:</strong> ${env.EXECUTION_TIME}</p>
                        <p><strong>Ambiente:</strong> ${params.AMBIENTE}</p>
                        <p><strong>Tipo de ejecución:</strong> Colección completa</p>
                        
                        <h3>Resumen:</h3>
                        <pre style="background-color: #f5f5f5; padding: 15px; border-radius: 5px; font-size: 12px;">
${summaryContent}
                        </pre>
                        
                        <p><strong>Reportes adjuntos:</strong> ${hasReports == 'YES' ? 'Sí' : 'No'}</p>
                        
                        <hr>
                        <p style="font-size: 11px; color: #666;"><em>Generado automáticamente por Jenkins</em></p>
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
                            attachmentsPattern: "${env.EXECUTION_FOLDER}\\**\\*.html",
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
                
                try {
                    archiveArtifacts artifacts: "${env.EXECUTION_FOLDER}\\**\\*", allowEmptyArchive: true
                    echo "[OK] Artefactos archivados"
                } catch (Exception e) {
                    echo "[WARN] No se archivaron: ${e.message}"
                }
            }
        }
        success {
            echo '[OK] Pipeline completado exitosamente'
        }
        failure {
            echo '[ERROR] Pipeline falló'
        }
    }
}