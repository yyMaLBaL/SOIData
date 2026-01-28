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
        
        stage('Instalar Reporters de Newman') {
            steps {
                powershell '''
                    Write-Host "Instalando reporters de Newman..."
                    npm install -g newman-reporter-html
                    npm install -g newman-reporter-json
                    Write-Host "Reporters instalados"
                '''
            }
        }
        
        stage('Validar Newman') {
            steps {
                powershell '''
                    Write-Host "Verificando version de Node..."
                    node -v
                    Write-Host "Verificando version de Newman..."
                    newman -v
                    Write-Host "Verificando reporters..."
                    newman --help | findstr reporter
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
                    
                    def basePath = "Ejecuciones/ACHDATA"
                    def executionFolder = "${basePath}/Fecha_${dateTimeOutput.split('_')[0]}_Hora_${dateTimeOutput.split('_')[1]}"
                    
                    powershell """
                        Write-Host "Creando estructura de carpetas..."
                        
                        if (!(Test-Path "${basePath}")) {
                            New-Item -ItemType Directory -Force -Path "${basePath}"
                            Write-Host "Carpeta base creada: ${basePath}"
                        }
                        
                        New-Item -ItemType Directory -Force -Path "${executionFolder}"
                        Write-Host "Carpeta de ejecucion creada: ${executionFolder}"
                        
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
        
        stage('Explorar Coleccion Postman') {
            steps {
                powershell '''
                    Write-Host "=== EXPLORANDO COLECCION ==="
                    
                    if (Test-Path "Collection/ACHDATA - YY.postman_collection.json") {
                        Write-Host "Coleccion encontrada"
                        
                        # Mostrar estructura de la coleccion
                        $collection = Get-Content "Collection/ACHDATA - YY.postman_collection.json" -Raw | ConvertFrom-Json
                        
                        Write-Host "Nombre coleccion: $($collection.info.name)"
                        Write-Host ""
                        
                        # Mostrar carpetas disponibles
                        if ($collection.item) {
                            Write-Host "CARPETAS DISPONIBLES:"
                            Write-Host "----------------------"
                            function Get-Folders($items, $prefix = "") {
                                foreach ($item in $items) {
                                    if ($item.item) {
                                        Write-Host "$($prefix)$($item.name)"
                                        Get-Folders $item.item "$prefix  "
                                    }
                                }
                            }
                            Get-Folders $collection.item
                        } else {
                            Write-Host "No se encontraron carpetas en la coleccion"
                        }
                    } else {
                        Write-Host "[ERROR] No se encuentra la coleccion"
                        Write-Host "Directorio actual: $(Get-Location)"
                        Write-Host "Archivos en Collection/:"
                        Get-ChildItem "Collection" -ErrorAction SilentlyContinue | ForEach-Object { Write-Host "  $($_.Name)" }
                    }
                    
                    Write-Host ""
                    Write-Host "=== ARCHIVOS EN ENVIRONMENT ==="
                    Get-ChildItem "Environment" -ErrorAction SilentlyContinue | ForEach-Object { Write-Host "  $($_.Name)" }
                '''
            }
        }
        
        stage('Ejecutar Coleccion Postman') {
            steps {
                script {
                    // Primero, obtener las carpetas reales de la coleccion
                    def folderStructure = powershell(returnStdout: true, script: '''
                        $folders = @()
                        if (Test-Path "Collection/ACHDATA - YY.postman_collection.json") {
                            $collection = Get-Content "Collection/ACHDATA - YY.postman_collection.json" -Raw | ConvertFrom-Json
                            
                            function Get-FolderNames($items) {
                                foreach ($item in $items) {
                                    if ($item.item) {
                                        $folders += $item.name
                                        Get-FolderNames $item.item
                                    }
                                }
                            }
                            Get-FolderNames $collection.item
                        }
                        Write-Output ($folders -join "||")
                    ''').trim()
                    
                    echo "Carpetas encontradas en coleccion: ${folderStructure}"
                    
                    def carpetasReales = folderStructure ? folderStructure.split('\\|\\|') : []
                    
                    // Definir ambientes a ejecutar
                    def ambientes = params.AMBIENTE == 'ALL' ? 
                        ['AT', 'SS', 'CS', 'CER', 'CERCS'] : 
                        [params.AMBIENTE]
                    
                    ambientes.each { ambiente ->
                        carpetasReales.each { carpetaReal ->
                            // Verificar si la carpeta comienza con el ambiente
                            if (carpetaReal.startsWith("${ambiente} ") || carpetaReal.contains("${ambiente}")) {
                                stage("${carpetaReal}") {
                                    def exitCode = powershell(returnStatus: true, script: """
                                        \$carpetaReal = "${carpetaReal}"
                                        \$ambiente = "${ambiente}"
                                        \$executionFolder = "${env.EXECUTION_FOLDER}".Replace('/', '\\\\')
                                        \$safeFileName = "\${carpetaReal}".Replace(' ', '_').Replace('-', '_').Replace('/', '_').Replace('\\\\', '_')
                                        \$reportFile = "\$executionFolder\\\\Reportes_HTML\\\\\${safeFileName}.html"
                                        \$jsonReport = "\$executionFolder\\\\Reportes_JSON\\\\\${safeFileName}.json"
                                        \$resultadosFile = "\$executionFolder\\\\resultados.csv"
                                        
                                        Write-Host "========================================="
                                        Write-Host "Ejecutando: \${carpetaReal}"
                                        Write-Host "========================================="
                                        
                                        \$newmanExitCode = 0
                                        \$outputText = ""
                                        
                                        try {
                                            Write-Host "Buscando coleccion..."
                                            \$collectionPath = "Collection/ACHDATA - YY.postman_collection.json"
                                            \$envPath = "Environment/ACHData QA.postman_environment.json"
                                            
                                            if (!(Test-Path \$collectionPath)) {
                                                Write-Host "[ERROR] No se encuentra: \$collectionPath"
                                                exit 1
                                            }
                                            
                                            if (!(Test-Path \$envPath)) {
                                                Write-Host "[ERROR] No se encuentra: \$envPath"
                                                exit 1
                                            }
                                            
                                            Write-Host "Ejecutando Newman..."
                                            \$outputText = newman run "\$collectionPath" `
                                                -e "\$envPath" `
                                                --folder "\${carpetaReal}" `
                                                --insecure `
                                                --reporters cli,html,json `
                                                --reporter-html-export "\${reportFile}" `
                                                --reporter-json-export "\${jsonReport}" 2>&1 | Out-String
                                            
                                            \$newmanExitCode = \$LASTEXITCODE
                                            Write-Host \$outputText
                                            
                                        } catch {
                                            Write-Host "Error: \$_"
                                            \$newmanExitCode = 1
                                        }
                                        
                                        if (Test-Path "\${jsonReport}") {
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
                                                
                                                "\${ambiente},\${carpetaReal},EJECUTADO,\${totalRequests},\${passedRequests},\${failedRequests},\${totalAssertions},\${passedAssertions},\${failedAssertions}" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                            } catch {
                                                Write-Host "Error procesando JSON: \$_"
                                                if (!(Test-Path "\${resultadosFile}")) {
                                                    "AMBIENTE,CARPETA,ESTADO,TOTAL_REQUESTS,REQUESTS_OK,REQUESTS_FAIL,TOTAL_ASSERTIONS,ASSERTIONS_OK,ASSERTIONS_FAIL" | Out-File -FilePath "\${resultadosFile}" -Encoding UTF8
                                                }
                                                "\${ambiente},\${carpetaReal},ERROR,0,0,0,0,0,0" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                            }
                                        } else {
                                            Write-Host "[INFO] No se genero reporte JSON"
                                            if (!(Test-Path "\${resultadosFile}")) {
                                                "AMBIENTE,CARPETA,ESTADO,TOTAL_REQUESTS,REQUESTS_OK,REQUESTS_FAIL,TOTAL_ASSERTIONS,ASSERTIONS_OK,ASSERTIONS_FAIL" | Out-File -FilePath "\${resultadosFile}" -Encoding UTF8
                                            }
                                            "\${ambiente},\${carpetaReal},NO_EJECUTADO,0,0,0,0,0,0" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                        }
                                        
                                        exit 0
                                    """)
                                    
                                    if (exitCode != 0) {
                                        echo "[WARN] Hubo un problema con ${carpetaReal}"
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
        
        stage('Ejecutar Coleccion Completa (alternativa)') {
            when {
                expression { 
                    def resultadosFile = "${env.EXECUTION_FOLDER}/resultados.csv"
                    def file = new File(resultadosFile)
                    return !file.exists() || file.text.trim().split('\n').size() <= 1
                }
            }
            steps {
                script {
                    echo "No se ejecutaron carpetas especificas. Ejecutando coleccion completa..."
                    
                    def exitCode = powershell(returnStatus: true, script: """
                        \$executionFolder = "${env.EXECUTION_FOLDER}".Replace('/', '\\\\')
                        \$ambiente = "${params.AMBIENTE}"
                        \$reportFile = "\$executionFolder\\\\Reportes_HTML\\\\coleccion_completa.html"
                        \$jsonReport = "\$executionFolder\\\\Reportes_JSON\\\\coleccion_completa.json"
                        \$resultadosFile = "\$executionFolder\\\\resultados.csv"
                        
                        Write-Host "========================================="
                        Write-Host "Ejecutando coleccion completa"
                        Write-Host "========================================="
                        
                        try {
                            \$outputText = newman run "Collection/ACHDATA - YY.postman_collection.json" `
                                -e "Environment/ACHData QA.postman_environment.json" `
                                --insecure `
                                --reporters cli,html,json `
                                --reporter-html-export "\${reportFile}" `
                                --reporter-json-export "\${jsonReport}" 2>&1 | Out-String
                            
                            Write-Host \$outputText
                            
                            if (Test-Path "\${jsonReport}") {
                                \$jsonContent = Get-Content "\${jsonReport}" -Raw | ConvertFrom-Json
                                \$totalRequests = \$jsonContent.run.stats.requests.total
                                \$failedRequests = \$jsonContent.run.stats.requests.failed
                                \$passedRequests = \$totalRequests - \$failedRequests
                                
                                \$totalAssertions = \$jsonContent.run.stats.assertions.total
                                \$failedAssertions = \$jsonContent.run.stats.assertions.failed
                                \$passedAssertions = \$totalAssertions - \$failedAssertions
                                
                                if (!(Test-Path "\${resultadosFile}")) {
                                    "AMBIENTE,CARPETA,ESTADO,TOTAL_REQUESTS,REQUESTS_OK,REQUESTS_FAIL,TOTAL_ASSERTIONS,ASSERTIONS_OK,ASSERTIONS_FAIL" | Out-File -FilePath "\${resultadosFile}" -Encoding UTF8
                                }
                                
                                "\${ambiente},COLECCION_COMPLETA,EJECUTADO,\${totalRequests},\${passedRequests},\${failedRequests},\${totalAssertions},\${passedAssertions},\${failedAssertions}" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                            }
                            
                        } catch {
                            Write-Host "Error: \$_"
                        }
                        
                        exit 0
                    """)
                }
            }
        }
        
        stage('Generar Resumen') {
            steps {
                powershell """
                    \$executionFolder = "${env.EXECUTION_FOLDER}".Replace('/', '\\\\')
                    \$summaryFile = "\$executionFolder\\\\resumen_ejecucion.txt"
                    \$resultadosFile = "\$executionFolder\\\\resultados.csv"
                    
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
                        \$noEjecutados = 0
                        \$errores = 0
                        \$totalRequests = 0
                        \$totalPassedRequests = 0
                        \$totalFailedRequests = 0
                        
                        foreach (\$line in \$lines) {
                            if (\$line -match "^AMBIENTE,") {
                                continue
                            }
                            
                            \$parts = \$line.Split(',')
                            if (\$parts.Count -ge 9) {
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
                                } elseif (\$estado -eq "NO_EJECUTADO") {
                                    \$noEjecutados++
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
                        "Carpetas no ejecutadas: \$noEjecutados" | Out-File -FilePath \$summaryFile -Append
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
                    
                    Write-Host ""
                    Write-Host "=== RESUMEN ==="
                    Get-Content \$summaryFile
                """
            }
        }
        
        stage('Enviar Reportes por Correo') {
            steps {
                script {
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