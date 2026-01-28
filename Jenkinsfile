pipeline {
    agent any
    
    parameters {
        choice(
            name: 'AMBIENTE',
            choices: ['AT', 'SS', 'CS', 'CER', 'CERCS'],
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
        
        stage('Listar Estructura de Coleccion') {
            steps {
                script {
                    echo "========================================="
                    echo "ESTRUCTURA COMPLETA DE LA COLECCION"
                    echo "========================================="
                    
                    powershell '''
                        $collection = Get-Content "Collection/ACHDATA - YY.postman_collection.json" -Raw | ConvertFrom-Json
                        
                        Write-Host "`nNombre de la coleccion: $($collection.info.name)"
                        Write-Host "`n========================================="
                        Write-Host "CARPETAS Y SUBCARPETAS:"
                        Write-Host "=========================================`n"
                        
                        function Show-Items {
                            param($items, $indent = "")
                            
                            foreach ($item in $items) {
                                Write-Host "$indent$($item.name)"
                                
                                if ($item.item) {
                                    Show-Items -items $item.item -indent "$indent  "
                                }
                            }
                        }
                        
                        if ($collection.item) {
                            Show-Items -items $collection.item
                        }
                        
                        Write-Host "`n========================================="
                    '''
                }
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
                    
                    def executionFolder = "Ejecuciones/ACHDATA/Fecha_${dateTimeOutput.replace('_', '_Hora_')}"
                    
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
                    // Lista de carpetas a buscar - ajusta estos nombres según lo que veas en el stage anterior
                    def carpetas = [
                        "${params.AMBIENTE} - Autenticacion",
                        "${params.AMBIENTE} - Detallada Natural Y Juridica",
                        "${params.AMBIENTE} - Usuarios",
                        "${params.AMBIENTE} - Roles",
                        "${params.AMBIENTE} - Carga Masiva de Usuarios"
                    ]
                    
                    carpetas.each { folderName ->
                        stage("Ejecutar: ${folderName}") {
                            def exitCode = powershell(returnStatus: true, script: """
                                \$folderName = "${folderName}"
                                \$executionFolder = "${env.EXECUTION_FOLDER}".Replace('/', '\\')
                                \$safeFileName = "${folderName}".Replace(' ', '_').Replace('-', '')
                                \$reportFile = "\$executionFolder\\report_\${safeFileName}.html"
                                \$jsonReport = "\$executionFolder\\report_\${safeFileName}.json"
                                \$resultadosFile = "\$executionFolder\\resultados.csv"
                                
                                Write-Host "========================================="
                                Write-Host "Intentando ejecutar: \${folderName}"
                                Write-Host "========================================="
                                
                                \$newmanExitCode = 0
                                \$folderExists = \$false
                                \$outputText = ""
                                
                                try {
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
                                        Write-Host "[INFO] Carpeta '\${folderName}' no encontrada en la coleccion"
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
                                        
                                        "\${folderName},EJECUTADO,\${totalRequests},\${passedRequests},\${failedRequests},\${totalAssertions},\${passedAssertions},\${failedAssertions}" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                    } catch {
                                        Write-Host "Error procesando JSON: \$_"
                                        "\${folderName},ERROR,0,0,0,0,0,0" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                    }
                                } else {
                                    Write-Host "[INFO] Carpeta no disponible"
                                    "\${folderName},NO_DISPONIBLE,0,0,0,0,0,0" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
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
                    "=========================================" | Out-File -FilePath \$summaryFile -Append
                    "" | Out-File -FilePath \$summaryFile -Append
                    
                    if (Test-Path "\$resultadosFile") {
                        "RESULTADOS:" | Out-File -FilePath \$summaryFile -Append
                        "-----------------------" | Out-File -FilePath \$summaryFile -Append
                        
                        \$lines = Get-Content "\$resultadosFile"
                        \$ejecutados = 0
                        \$noDisponibles = 0
                        \$errores = 0
                        
                        foreach (\$line in \$lines) {
                            \$parts = \$line.Split(',')
                            if (\$parts.Count -eq 8) {
                                \$folder = \$parts[0].Trim()
                                \$estado = \$parts[1].Trim()
                                
                                if (\$estado -eq "EJECUTADO") {
                                    \$ejecutados++
                                    \$reqTotal = \$parts[2]
                                    \$reqPassed = \$parts[3]
                                    \$reqFailed = \$parts[4]
                                    "[OK] \$folder" | Out-File -FilePath \$summaryFile -Append
                                    "     Requests: \$reqTotal (OK:\$reqPassed / FAIL:\$reqFailed)" | Out-File -FilePath \$summaryFile -Append
                                } elseif (\$estado -eq "NO_DISPONIBLE") {
                                    \$noDisponibles++
                                    "[N/A] \$folder" | Out-File -FilePath \$summaryFile -Append
                                } else {
                                    \$errores++
                                    "[ERROR] \$folder" | Out-File -FilePath \$summaryFile -Append
                                }
                                "" | Out-File -FilePath \$summaryFile -Append
                            }
                        }
                        
                        "=========================================" | Out-File -FilePath \$summaryFile -Append
                        "RESUMEN:" | Out-File -FilePath \$summaryFile -Append
                        "Ejecutados: \$ejecutados" | Out-File -FilePath \$summaryFile -Append
                        "No disponibles: \$noDisponibles" | Out-File -FilePath \$summaryFile -Append
                        "Errores: \$errores" | Out-File -FilePath \$summaryFile -Append
                    }
                    
                    Write-Host ""
                    Write-Host "=== RESUMEN ==="
                    Get-Content \$summaryFile
                """
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
                    echo "[WARN] No se archivaron artefactos: ${e.message}"
                }
            }
        }
        success {
            echo '[OK] Pipeline completado'
        }
    }
}