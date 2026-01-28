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
                    
                    def executionFolder = "Executions/ACHDATA/Fecha_${dateTimeOutput.replace('_', '_Hora_')}"
                    
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
        
        stage('Ejecutar Coleccion Postman') {
            steps {
                script {
                    // Obtener las carpetas disponibles del JSON
                    def carpetasDisponibles = powershell(returnStdout: true, script: '''
                        $collection = Get-Content "Collection/ACHDATA - YY.postman_collection.json" -Raw -Encoding UTF8 | ConvertFrom-Json
                        
                        function Get-FolderNames {
                            param($items)
                            $folders = @()
                            foreach ($item in $items) {
                                if ($item.name -match "^(AT|SS|CS|CER|CERCS) - ") {
                                    $folders += $item.name
                                }
                                if ($item.item) {
                                    $folders += Get-FolderNames -items $item.item
                                }
                            }
                            return $folders
                        }
                        
                        $allFolders = Get-FolderNames -items $collection.item
                        $allFolders | ForEach-Object { [Console]::OutputEncoding = [System.Text.Encoding]::UTF8; Write-Output $_ }
                    ''').trim().split('\n').collect { it.trim() }.findAll { it }
                    
                    echo "Carpetas encontradas: ${carpetasDisponibles.size()}"
                    carpetasDisponibles.each { echo "  - ${it}" }
                    
                    // Definir ambientes a ejecutar
                    def ambientes = params.AMBIENTE == 'ALL' ? 
                        ['AT', 'SS', 'CS', 'CER', 'CERCS'] : 
                        [params.AMBIENTE]
                    
                    // Filtrar carpetas por ambiente
                    ambientes.each { ambiente ->
                        def carpetasParaEsteAmbiente = carpetasDisponibles.findAll { it.startsWith("${ambiente} - ") }
                        
                        carpetasParaEsteAmbiente.each { folderName ->
                            // Extraer el nombre sin el prefijo del ambiente
                            def carpetaBase = folderName.replaceFirst("${ambiente} - ", "")
                            
                            stage("${folderName}") {
                                def exitCode = powershell(returnStatus: true, script: """
                                    \$OutputEncoding = [Console]::OutputEncoding = [System.Text.Encoding]::UTF8
                                    chcp 65001 > \$null
                                    
                                    \$folderName = "${folderName}"
                                    \$ambiente = "${ambiente}"
                                    \$carpetaBase = "${carpetaBase}"
                                    \$executionFolder = "${env.EXECUTION_FOLDER}".Replace('/', '\\')
                                    
                                    if (-not (Test-Path "\$executionFolder")) {
                                        New-Item -ItemType Directory -Force -Path "\$executionFolder" | Out-Null
                                    }
                                    
                                    \$safeFileName = ("\${ambiente}_\${carpetaBase}" -replace '[^a-zA-Z0-9_-]', '_')
                                    \$reportFile = "\$executionFolder\\report_\${safeFileName}.html"
                                    \$jsonReport = "\$executionFolder\\report_\${safeFileName}.json"
                                    \$resultadosFile = "\$executionFolder\\resultados.csv"
                                    
                                    Write-Host "========================================="
                                    Write-Host "Ejecutando: \${folderName}"
                                    Write-Host "========================================="
                                    
                                    \$newmanExitCode = 0
                                    \$folderExists = \$false
                                    
                                    try {
                                        newman run "Collection/ACHDATA - YY.postman_collection.json" `
                                            -e "Environment/ACHData QA.postman_environment.json" `
                                            --folder "\${folderName}" `
                                            --insecure `
                                            --reporters cli,html,json `
                                            --reporter-html-export "\${reportFile}" `
                                            --reporter-json-export "\${jsonReport}"
                                        
                                        \$newmanExitCode = \$LASTEXITCODE
                                        
                                        if (\$newmanExitCode -eq 0 -and (Test-Path "\${jsonReport}")) {
                                            \$folderExists = \$true
                                        } else {
                                            \$folderExists = \$false
                                        }
                                        
                                    } catch {
                                        Write-Host "Error: \$_"
                                        \$folderExists = \$false
                                    }
                                    
                                    if (\$folderExists -and (Test-Path "\${jsonReport}")) {
                                        try {
                                            \$jsonContent = Get-Content "\${jsonReport}" -Raw -Encoding UTF8 | ConvertFrom-Json
                                            \$totalRequests = \$jsonContent.run.stats.requests.total
                                            \$failedRequests = \$jsonContent.run.stats.requests.failed
                                            \$passedRequests = \$totalRequests - \$failedRequests
                                            
                                            \$totalAssertions = \$jsonContent.run.stats.assertions.total
                                            \$failedAssertions = \$jsonContent.run.stats.assertions.failed
                                            \$passedAssertions = \$totalAssertions - \$failedAssertions
                                            
                                            Write-Host ""
                                            Write-Host "[OK] Requests: \${totalRequests} (OK:\${passedRequests} / FAIL:\${failedRequests})"
                                            Write-Host "[OK] Assertions: \${totalAssertions} (OK:\${passedAssertions} / FAIL:\${failedAssertions})"
                                            
                                            "\${ambiente},\${carpetaBase},EJECUTADO,\${totalRequests},\${passedRequests},\${failedRequests},\${totalAssertions},\${passedAssertions},\${failedAssertions}" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                        } catch {
                                            Write-Host "Error procesando JSON: \$_"
                                            "\${ambiente},\${carpetaBase},ERROR,0,0,0,0,0,0" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                        }
                                    } else {
                                        Write-Host "[INFO] Carpeta no disponible"
                                        "\${ambiente},\${carpetaBase},NO_DISPONIBLE,0,0,0,0,0,0" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                    }
                                    
                                    exit 0
                                """)
                            }
                        }
                    }
                }
            }
        }
        
        stage('Generar Resumen') {
            steps {
                powershell """
                    \$OutputEncoding = [System.Text.Encoding]::UTF8
                    
                    \$executionFolder = "${env.EXECUTION_FOLDER}".Replace('/', '\\')
                    \$summaryFile = "\$executionFolder\\resumen_ejecucion.txt"
                    \$resultadosFile = "\$executionFolder\\resultados.csv"
                    
                    "=========================================" | Out-File -FilePath \$summaryFile -Encoding UTF8
                    "RESUMEN DE EJECUCION - ACHDATA" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                    "=========================================" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                    "Fecha: ${env.EXECUTION_DATE}" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                    "Hora: ${env.EXECUTION_TIME}" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                    "Ambiente: ${params.AMBIENTE}" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                    "=========================================" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                    "" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                    
                    if (Test-Path "\$resultadosFile") {
                        "RESULTADOS:" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                        "-----------------------" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                        "" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                        
                        \$lines = Get-Content "\$resultadosFile" -Encoding UTF8
                        \$ejecutados = 0
                        \$noDisponibles = 0
                        \$errores = 0
                        \$totalRequests = 0
                        \$totalPassedRequests = 0
                        \$totalFailedRequests = 0
                        
                        foreach (\$line in \$lines) {
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
                                        "[OK] \$amb - \$carpeta" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                                    } else {
                                        "[WARN] \$amb - \$carpeta" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                                    }
                                    "     Requests: \$reqTotal (OK:\$reqPassed / FAIL:\$reqFailed)" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                                } elseif (\$estado -eq "NO_DISPONIBLE") {
                                    \$noDisponibles++
                                    "[N/A] \$amb - \$carpeta" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                                } else {
                                    \$errores++
                                    "[ERROR] \$amb - \$carpeta" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                                }
                                "" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                            }
                        }
                        
                        "=========================================" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                        "RESUMEN:" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                        "-----------------------" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                        "Carpetas ejecutadas: \$ejecutados" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                        "Carpetas no disponibles: \$noDisponibles" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                        "Carpetas con error: \$errores" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                        "" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                        
                        if (\$ejecutados -gt 0) {
                            "TOTALES:" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                            "Requests totales: \$totalRequests" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                            "Requests exitosos: \$totalPassedRequests" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                            "Requests fallidos: \$totalFailedRequests" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                            
                            if (\$totalRequests -gt 0) {
                                \$successRate = [math]::Round((\$totalPassedRequests / \$totalRequests) * 100, 2)
                                "Tasa de exito: \$successRate%" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                            }
                        }
                    } else {
                        "No se encontraron resultados" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                    }
                    
                    \$htmlReports = Get-ChildItem "\$executionFolder" -Filter "*.html" -ErrorAction SilentlyContinue
                    "" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                    "=========================================" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                    "Reportes HTML generados: \$(\$htmlReports.Count)" | Out-File -FilePath \$summaryFile -Append -Encoding UTF8
                    
                    Write-Host ""
                    Write-Host "=== RESUMEN ==="
                    Get-Content \$summaryFile -Encoding UTF8
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
                        summaryContent = "No se pudo leer el resumen"
                    }
                    
                    def emailBody = """
                    <html>
                    <body style="font-family: Arial, sans-serif;">
                        <h2>Reporte de Ejecucion Postman - ACHDATA</h2>
                        <p><strong>Fecha:</strong> ${env.EXECUTION_DATE} a las ${env.EXECUTION_TIME}</p>
                        <p><strong>Ambiente:</strong> ${params.AMBIENTE}</p>
                        
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
                            attachmentsPattern: "${env.EXECUTION_FOLDER}/**/*.html",
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