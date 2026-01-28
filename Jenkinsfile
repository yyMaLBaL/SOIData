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
                    if (Test-Path "Collection/ACHDATA - YY.postman_collection.json") {
                        Write-Host "[OK] Colección encontrada"
                        $collPath = Resolve-Path "Collection/ACHDATA - YY.postman_collection.json"
                        Write-Host "Ruta: $collPath"
                    } else {
                        Write-Host "[ERROR] Colección NO encontrada"
                        Write-Host "Buscando archivos..."
                        Get-ChildItem -Recurse -Filter "*.json" | Select-Object Name, Directory | Format-Table
                    }
                    
                    Write-Host ""
                    
                    # Verificar entorno
                    if (Test-Path "Environment/ACHData QA.postman_environment.json") {
                        Write-Host "[OK] Entorno encontrado"
                        $envPath = Resolve-Path "Environment/ACHData QA.postman_environment.json"
                        Write-Host "Ruta: $envPath"
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
                powershell '''
                    Write-Host "=== EXTRAYENDO CARPETAS DE LA COLECCIÓN ==="
                    
                    $collectionPath = "Collection/ACHDATA - YY.postman_collection.json"
                    
                    if (Test-Path $collectionPath) {
                        # Leer y analizar el JSON
                        $jsonContent = Get-Content $collectionPath -Raw
                        
                        # Usar expresiones regulares para extraer nombres de carpetas
                        # Buscar patrones de nombres en el JSON
                        $pattern = '"name"\s*:\s*"([^"]+)"'
                        $matches = [regex]::Matches($jsonContent, $pattern)
                        
                        Write-Host "Nombres encontrados en la colección:"
                        Write-Host "--------------------------------------"
                        
                        $uniqueNames = @{}
                        foreach ($match in $matches) {
                            $name = $match.Groups[1].Value
                            if (-not $uniqueNames.ContainsKey($name)) {
                                $uniqueNames[$name] = $true
                                Write-Host "- $name"
                            }
                        }
                        
                        Write-Host ""
                        Write-Host "Total nombres únicos: $($uniqueNames.Count)"
                        
                        # Guardar en archivo para referencia
                        $uniqueNames.Keys | Out-File "folder_names.txt" -Encoding UTF8
                    } else {
                        Write-Host "ERROR: No se encontró la colección"
                    }
                '''
            }
        }
        
        stage('Ejecutar Prueba Simple') {
            steps {
                powershell '''
                    Write-Host "=== EJECUTANDO PRUEBA SIMPLE ==="
                    
                    # Primero, ejecutar sin carpeta específica para ver qué hay
                    newman run "Collection/ACHDATA - YY.postman_collection.json" `
                        -e "Environment/ACHData QA.postman_environment.json" `
                        --insecure `
                        --reporters cli `
                        --reporter-cli-no-summary
                    
                    Write-Host ""
                    Write-Host "=== EJECUTANDO POR ITEM ==="
                    
                    # Obtener el primer item de la colección
                    $jsonContent = Get-Content "Collection/ACHDATA - YY.postman_collection.json" -Raw | ConvertFrom-Json
                    
                    if ($jsonContent.item -and $jsonContent.item.Count -gt 0) {
                        $firstItem = $jsonContent.item[0].name
                        Write-Host "Primer item encontrado: $firstItem"
                        
                        Write-Host ""
                        Write-Host "Ejecutando: $firstItem"
                        newman run "Collection/ACHDATA - YY.postman_collection.json" `
                            -e "Environment/ACHData QA.postman_environment.json" `
                            --folder "$firstItem" `
                            --insecure `
                            --reporters cli `
                            --reporter-cli-no-summary
                    }
                '''
            }
        }
        
        stage('Ejecutar Coleccion Completa') {
            steps {
                script {
                    // Primero, obtener la lista real de carpetas de la colección
                    def folders = powershell(returnStdout: true, script: '''
                        $collectionPath = "Collection/ACHDATA - YY.postman_collection.json"
                        $folders = @()
                        
                        if (Test-Path $collectionPath) {
                            $jsonContent = Get-Content $collectionPath -Raw | ConvertFrom-Json
                            
                            function Get-FolderNames {
                                param($items)
                                
                                foreach ($item in $items) {
                                    if ($item.name) {
                                        $folders += $item.name
                                    }
                                    
                                    if ($item.item) {
                                        $folders += Get-FolderNames -items $item.item
                                    }
                                }
                                
                                return $folders
                            }
                            
                            $folders = Get-FolderNames -items $jsonContent.item
                        }
                        
                        # Filtrar solo carpetas (excluir requests individuales)
                        # Asumiendo que las carpetas son las que empiezan con los códigos de ambiente
                        $filteredFolders = $folders | Where-Object { 
                            $_ -match '^(AT|SS|CS|CER|CERCS)\s*-'
                        }
                        
                        Write-Output ($filteredFolders -join "|")
                    ''').trim().split('\\|')
                    
                    echo "Carpetas encontradas: ${folders}"
                    
                    if (folders.size() == 0) {
                        echo "[WARN] No se encontraron carpetas con el patrón esperado"
                        echo "Intentando ejecutar toda la colección..."
                        
                        stage("Ejecutar Coleccion Completa") {
                            powershell """
                                \$executionFolder = "${env.EXECUTION_FOLDER}"
                                \$reportFile = "\$executionFolder\\report_completo.html"
                                \$jsonReport = "\$executionFolder\\report_completo.json"
                                
                                Write-Host "=== EJECUTANDO COLECCIÓN COMPLETA ==="
                                
                                newman run "Collection/ACHDATA - YY.postman_collection.json" `
                                    -e "Environment/ACHData QA.postman_environment.json" `
                                    --insecure `
                                    --reporters cli,html,json `
                                    --reporter-html-export "\$reportFile" `
                                    --reporter-json-export "\$jsonReport"
                                
                                if (\$LASTEXITCODE -eq 0) {
                                    Write-Host "[OK] Ejecución completada"
                                } else {
                                    Write-Host "[WARN] Ejecución con errores"
                                }
                            """
                        }
                    } else {
                        // Filtrar por ambiente seleccionado
                        def ambientes = params.AMBIENTE == 'ALL' ? 
                            ['AT', 'SS', 'CS', 'CER', 'CERCS'] : 
                            [params.AMBIENTE]
                        
                        folders.each { folder ->
                            def ambiente = folder.split(' - ')[0].trim()
                            
                            if (ambientes.contains(ambiente) || params.AMBIENTE == 'ALL') {
                                stage("${folder}") {
                                    powershell """
                                        \$folderName = "${folder}"
                                        \$ambiente = "${ambiente}"
                                        \$executionFolder = "${env.EXECUTION_FOLDER}"
                                        \$safeFileName = "\${folder}".Replace(' ', '_').Replace('/', '_').Replace('\\', '_')
                                        \$reportFile = "\$executionFolder\\report_\${safeFileName}.html"
                                        \$jsonReport = "\$executionFolder\\report_\${safeFileName}.json"
                                        \$resultadosFile = "\$executionFolder\\resultados.csv"
                                        
                                        Write-Host "========================================="
                                        Write-Host "Ejecutando: \${folderName}"
                                        Write-Host "========================================="
                                        
                                        newman run "Collection/ACHDATA - YY.postman_collection.json" `
                                            -e "Environment/ACHData QA.postman_environment.json" `
                                            --folder "\${folderName}" `
                                            --insecure `
                                            --reporters cli,html,json `
                                            --reporter-html-export "\$reportFile" `
                                            --reporter-json-export "\$jsonReport"
                                        
                                        if (Test-Path "\${jsonReport}") {
                                            try {
                                                \$jsonContent = Get-Content "\${jsonReport}" -Raw | ConvertFrom-Json
                                                \$totalRequests = \$jsonContent.run.stats.requests.total
                                                \$failedRequests = \$jsonContent.run.stats.requests.failed
                                                \$passedRequests = \$totalRequests - \$failedRequests
                                                
                                                "\${folderName},EJECUTADO,\${totalRequests},\${passedRequests},\${failedRequests}" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                                Write-Host "[OK] Resultados guardados"
                                            } catch {
                                                Write-Host "Error procesando JSON"
                                                "\${folderName},ERROR,0,0,0" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                            }
                                        }
                                    """
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
                    \$executionFolder = "${env.EXECUTION_FOLDER}"
                    \$summaryFile = "\$executionFolder\\resumen_ejecucion.txt"
                    
                    "=========================================" | Out-File -FilePath \$summaryFile
                    "RESUMEN DE EJECUCION - ACHDATA" | Out-File -FilePath \$summaryFile -Append
                    "=========================================" | Out-File -FilePath \$summaryFile -Append
                    "Fecha: ${env.EXECUTION_DATE}" | Out-File -FilePath \$summaryFile -Append
                    "Hora: ${env.EXECUTION_TIME}" | Out-File -FilePath \$summaryFile -Append
                    "Ambiente: ${params.AMBIENTE}" | Out-File -FilePath \$summaryFile -Append
                    "=========================================" | Out-File -FilePath \$summaryFile -Append
                    
                    # Listar reportes generados
                    \$htmlReports = Get-ChildItem "\$executionFolder" -Filter "*.html" -ErrorAction SilentlyContinue
                    if (\$htmlReports) {
                        "Reportes HTML generados:" | Out-File -FilePath \$summaryFile -Append
                        foreach (\$report in \$htmlReports) {
                            "  - \$(\$report.Name)" | Out-File -FilePath \$summaryFile -Append
                        }
                    } else {
                        "No se generaron reportes HTML" | Out-File -FilePath \$summaryFile -Append
                    }
                    
                    Get-Content \$summaryFile
                """
            }
        }
        
        stage('Enviar Reportes') {
            steps {
                script {
                    emailext(
                        to: 'yeinerballesta@cbit-online.com',
                        subject: "Reporte Postman ACHDATA - ${params.AMBIENTE} - ${env.EXECUTION_DATE}",
                        body: """
                        <html>
                        <body>
                            <h2>Reporte de Ejecución Postman - ACHDATA</h2>
                            <p><strong>Fecha:</strong> ${env.EXECUTION_DATE}</p>
                            <p><strong>Hora:</strong> ${env.EXECUTION_TIME}</p>
                            <p><strong>Ambiente:</strong> ${params.AMBIENTE}</p>
                            <p><strong>Carpeta de ejecución:</strong> ${env.EXECUTION_FOLDER}</p>
                        </body>
                        </html>
                        """,
                        mimeType: 'text/html',
                        attachmentsPattern: "${env.EXECUTION_FOLDER}/**/*",
                        from: 'jenkins@cbit-online.com'
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