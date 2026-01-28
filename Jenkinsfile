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
        
        stage('Verificar Estructura') {
            steps {
                powershell '''
                    Write-Host "=== VERIFICANDO ESTRUCTURA ==="
                    Write-Host "Directorio actual: $(Get-Location)"
                    Write-Host ""
                    
                    # Verificar colección
                    $collectionPath = "Collection/ACHDATA - YY.postman_collection.json"
                    if (Test-Path $collectionPath) {
                        Write-Host "[OK] Colección encontrada"
                        $collPath = Resolve-Path $collectionPath
                        Write-Host "Ruta: $collPath"
                        Write-Host ""
                        Write-Host "Tamaño del archivo: $((Get-Item $collectionPath).Length) bytes"
                    } else {
                        Write-Host "[ERROR] Colección NO encontrada en: $collectionPath"
                        Write-Host ""
                        Write-Host "Buscando archivos de colección..."
                        Get-ChildItem -Recurse -Filter "*.json" | Where-Object { $_.Name -match "postman" } | Select-Object Name, FullName | Format-Table
                    }
                    
                    Write-Host ""
                    
                    # Verificar entorno
                    $envPath = "Environment/ACHData QA.postman_environment.json"
                    if (Test-Path $envPath) {
                        Write-Host "[OK] Entorno encontrado"
                        $envResolved = Resolve-Path $envPath
                        Write-Host "Ruta: $envResolved"
                    } else {
                        Write-Host "[ERROR] Entorno NO encontrado en: $envPath"
                        Write-Host ""
                        Write-Host "Buscando archivos de entorno..."
                        Get-ChildItem -Recurse -Filter "*environment*.json" | Select-Object Name, FullName | Format-Table
                    }
                '''
            }
        }
        
        stage('Listar Contenido Coleccion') {
            steps {
                powershell '''
                    Write-Host "=== ANALIZANDO ESTRUCTURA DE LA COLECCIÓN ==="
                    
                    $collectionPath = "Collection/ACHDATA - YY.postman_collection.json"
                    
                    if (Test-Path $collectionPath) {
                        # Mostrar las primeras líneas del archivo para ver su estructura
                        Write-Host "Primeras 10 líneas del archivo:"
                        Write-Host "--------------------------------"
                        Get-Content $collectionPath -First 10
                        
                        Write-Host ""
                        Write-Host "Últimas 5 líneas del archivo:"
                        Write-Host "--------------------------------"
                        Get-Content $collectionPath -Last 5
                        
                        Write-Host ""
                        Write-Host "=== INTENTANDO LEER COMO JSON ==="
                        
                        try {
                            # Intentar leer el JSON
                            $jsonContent = Get-Content $collectionPath -Raw | ConvertFrom-Json
                            
                            Write-Host "[OK] JSON parseado correctamente"
                            Write-Host "Nombre de la colección: $($jsonContent.info.name)"
                            Write-Host ""
                            
                            # Función recursiva para listar items
                            function List-Items {
                                param($items, $level = 0)
                                
                                $indent = "  " * $level
                                
                                foreach ($item in $items) {
                                    if ($item.name) {
                                        Write-Host "${indent}- $($item.name)"
                                        
                                        # Si es una carpeta (tiene subitems)
                                        if ($item.item) {
                                            List-Items -items $item.item -level ($level + 1)
                                        }
                                        
                                        # Si tiene request
                                        if ($item.request) {
                                            Write-Host "${indent}  [REQUEST] $($item.request.method) $($item.request.url.raw)"
                                        }
                                    }
                                }
                            }
                            
                            if ($jsonContent.item) {
                                Write-Host "Items en la colección:"
                                Write-Host "======================"
                                List-Items -items $jsonContent.item
                            } else {
                                Write-Host "La colección no tiene items definidos"
                            }
                            
                        } catch {
                            Write-Host "[ERROR] No se pudo parsear el JSON: $_"
                            Write-Host ""
                            Write-Host "Posible problema de encoding o formato"
                        }
                        
                    } else {
                        Write-Host "ERROR: Archivo no encontrado"
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
        
        stage('Probar Ejecucion Simple') {
            steps {
                script {
                    // Primero probar ejecutar una carpeta específica si sabemos su nombre
                    def testFolders = [
                        "Autenticación",
                        "Detallada Natural Y Juridica",
                        "AT - Autenticación",
                        "SS - Autenticación",
                        "CS - Autenticación",
                        "CER - Autenticación",
                        "CERCS - Autenticación"
                    ]
                    
                    testFolders.each { folderName ->
                        stage("Probar: ${folderName}") {
                            def result = powershell(returnStdout: true, script: """
                                \$folderName = "${folderName}"
                                \$executionFolder = "${env.EXECUTION_FOLDER}"
                                
                                Write-Host "Probando ejecutar carpeta: '\$folderName'"
                                Write-Host "----------------------------------------"
                                
                                # Intentar ejecutar
                                \$output = newman run "Collection/ACHDATA - YY.postman_collection.json" `
                                    -e "Environment/ACHData QA.postman_environment.json" `
                                    --folder "\$folderName" `
                                    --insecure `
                                    --reporters cli `
                                    --reporter-cli-no-summary 2>&1
                                
                                Write-Host "Salida:"
                                Write-Host \$output
                                
                                # Verificar si encontró la carpeta
                                if (\$output -match "Unable to find a folder") {
                                    Write-Host "RESULTADO: Carpeta no encontrada"
                                    Write-Output "NOT_FOUND"
                                } elseif (\$output -match "iteration") {
                                    Write-Host "RESULTADO: Ejecutado con éxito"
                                    Write-Output "SUCCESS"
                                } else {
                                    Write-Host "RESULTADO: Otro resultado"
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
                    \$executionFolder = "${env.EXECUTION_FOLDER}"
                    \$reportFile = "\$executionFolder\\report_completo.html"
                    \$jsonReport = "\$executionFolder\\report_completo.json"
                    \$resultadosFile = "\$executionFolder\\resultados.csv"
                    
                    Write-Host "=== EJECUTANDO COLECCIÓN COMPLETA ==="
                    Write-Host "Esto ejecutará TODOS los requests de la colección"
                    Write-Host ""
                    
                    # Ejecutar toda la colección
                    newman run "Collection/ACHDATA - YY.postman_collection.json" `
                        -e "Environment/ACHData QA.postman_environment.json" `
                        --insecure `
                        --reporters cli,html,json `
                        --reporter-html-export "\$reportFile" `
                        --reporter-json-export "\$jsonReport"
                    
                    \$exitCode = \$LASTEXITCODE
                    
                    if (Test-Path "\$jsonReport") {
                        try {
                            \$jsonContent = Get-Content "\$jsonReport" -Raw | ConvertFrom-Json
                            \$totalRequests = \$jsonContent.run.stats.requests.total
                            \$failedRequests = \$jsonContent.run.stats.requests.failed
                            \$passedRequests = \$totalRequests - \$failedRequests
                            
                            Write-Host ""
                            Write-Host "=== RESULTADOS ==="
                            Write-Host "Total requests: \$totalRequests"
                            Write-Host "Passed: \$passedRequests"
                            Write-Host "Failed: \$failedRequests"
                            
                            "COLECCION_COMPLETA,EJECUTADO,\$totalRequests,\$passedRequests,\$failedRequests" | Out-File -FilePath "\$resultadosFile" -Encoding UTF8
                            
                        } catch {
                            Write-Host "Error procesando JSON: \$_"
                        }
                    }
                    
                    if (\$exitCode -eq 0) {
                        Write-Host "[OK] Ejecución completada exitosamente"
                    } else {
                        Write-Host "[WARN] Ejecución completada con errores (código: \$exitCode)"
                    }
                """
            }
        }
        
        stage('Generar Resumen') {
            steps {
                powershell """
                    \$executionFolder = "${env.EXECUTION_FOLDER}"
                    \$summaryFile = "\$executionFolder\\resumen_ejecucion.txt"
                    
                    "=========================================" | Out-File -FilePath \$summaryFile
                    "RESUMEN DE EJECUCION - ACHDATA" | Out-File -FilePath \$summaryFile -Append
                    "=========================================" | Out-File -FilePath \$summaryFile -Append
                    "Fecha: ${env.EXECUTION_DATE}" | Out-File -FilePath \$summaryFile -Append
                    "Hora: ${env.EXECUTION_TIME}" | Out-File -FilePath \$summaryFile -Append
                    "Ambiente seleccionado: ${params.AMBIENTE}" | Out-File -FilePath \$summaryFile -Append
                    "=========================================" | Out-File -FilePath \$summaryFile -Append
                    "" | Out-File -FilePath \$summaryFile -Append
                    
                    # Listar reportes generados
                    \$htmlReports = Get-ChildItem "\$executionFolder" -Filter "*.html" -ErrorAction SilentlyContinue
                    if (\$htmlReports) {
                        "REPORTES GENERADOS:" | Out-File -FilePath \$summaryFile -Append
                        "-------------------" | Out-File -FilePath \$summaryFile -Append
                        foreach (\$report in \$htmlReports) {
                            "  - \$(\$report.Name)" | Out-File -FilePath \$summaryFile -Append
                        }
                        "" | Out-File -FilePath \$summaryFile -Append
                    } else {
                        "No se generaron reportes HTML" | Out-File -FilePath \$summaryFile -Append
                        "" | Out-File -FilePath \$summaryFile -Append
                    }
                    
                    # Mostrar resultados si existen
                    \$resultadosFile = "\$executionFolder\\resultados.csv"
                    if (Test-Path \$resultadosFile) {
                        "RESULTADOS:" | Out-File -FilePath \$summaryFile -Append
                        "----------" | Out-File -FilePath \$summaryFile -Append
                        Get-Content \$resultadosFile | Out-File -FilePath \$summaryFile -Append
                        "" | Out-File -FilePath \$summaryFile -Append
                    }
                    
                    Write-Host ""
                    Write-Host "=== RESUMEN FINAL ==="
                    Get-Content \$summaryFile
                """
            }
        }
        
        stage('Enviar Reportes por Correo') {
            steps {
                script {
                    // Verificar si hay reportes
                    def hasReports = powershell(returnStdout: true, script: """
                        \$executionFolder = "${env.EXECUTION_FOLDER}"
                        \$htmlFiles = Get-ChildItem "\$executionFolder" -Filter "*.html" -ErrorAction SilentlyContinue
                        if (\$htmlFiles) {
                            Write-Output "YES"
                        } else {
                            Write-Output "NO"
                        }
                    """).trim()
                    
                    def summaryContent = ""
                    try {
                        summaryContent = readFile("${env.EXECUTION_FOLDER}\\resumen_ejecucion.txt")
                    } catch (Exception e) {
                        summaryContent = "No se pudo leer el resumen"
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