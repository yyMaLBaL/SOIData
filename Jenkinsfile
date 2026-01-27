pipeline {
    agent any
    
    parameters {
        choice(
            name: 'AMBIENTE',
            choices: ['ALL', 'AT', 'SS', 'CS', 'CER', 'CERCS'],
            description: 'Selecciona el ambiente a ejecutar'
        )
        choice(
            name: 'SECCION',
            choices: [
                'TODAS',
                'Autenticacion',
                'Consultas',
                'Servicios ACH',
                'SDH',
                'Administracion',
                'Monitoreo'
            ],
            description: 'Selecciona la seccion a ejecutar'
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
                    // Mapa de nombres sin acentos a nombres reales en Postman
                    def nombresMapeados = [
                        'Autenticacion': 'Autenticación',
                        'Administracion': 'Administración'
                    ]
                    
                    // Definir todas las carpetas por seccion
                    def todasLasCarpetas = [
                        'Autenticacion': ['Autenticación'],
                        'Consultas': [
                            'Detallada Natural Y Jurídica',
                            'Consolidada Natural',
                            'Consolidada Jurídica',
                            'Extendida Natural',
                            'Masivas Con Filtros',
                            'Transferencias Y Pagos ACH Personas',
                            'Transferencias Y Pagos ACH Empresas',
                            'Contactabilidad Natural Y Jurídica',
                            'Parámetros Consultas',
                            'Prueba De Cobertura',
                            'Consulta por categoría - Persona Natural',
                            'Otras Consultas Masivas'
                        ],
                        'Servicios ACH': ['Servicios ACH'],
                        'SDH': ['SDH'],
                        'Administracion': [
                            'Estadísticas',
                            'Auditoría',
                            'Usuarios',
                            'Roles',
                            'Carga Masiva de Usuarios'
                        ],
                        'Monitoreo': ['Monitoreo']
                    ]
                    
                    // Determinar que carpetas ejecutar segun la seleccion
                    def carpetasAEjecutar = []
                    if (params.SECCION == 'TODAS') {
                        todasLasCarpetas.each { seccion, carpetas ->
                            carpetasAEjecutar.addAll(carpetas)
                        }
                    } else {
                        carpetasAEjecutar = todasLasCarpetas[params.SECCION]
                    }
                    
                    // Definir ambientes a ejecutar
                    def ambientes = params.AMBIENTE == 'ALL' ? 
                        ['AT', 'SS', 'CS', 'CER', 'CERCS'] : 
                        [params.AMBIENTE]
                    
                    echo "Carpetas a ejecutar: ${carpetasAEjecutar}"
                    echo "Ambientes a ejecutar: ${ambientes}"
                    
                    // Ejecutar cada combinacion
                    carpetasAEjecutar.each { carpeta ->
                        ambientes.each { ambiente ->
                            // Construir el nombre de la carpeta
                            def folderName = "${ambiente} - ${carpeta}"
                            
                            stage("${ambiente} - ${carpeta}") {
                                def exitCode = powershell(returnStatus: true, script: """
                                    \$folderName = "${folderName}"
                                    \$ambiente = "${ambiente}"
                                    \$carpeta = "${carpeta}"
                                    \$executionFolder = "${env.EXECUTION_FOLDER}".Replace('/', '\\')
                                    
                                    # Crear nombres de archivo seguros (sin espacios ni caracteres especiales)
                                    \$safeFileName = ("\${ambiente}_\${carpeta}" -replace ' ', '_' -replace '[^a-zA-Z0-9_-]', '')
                                    \$reportFile = "\$executionFolder\\report_\${safeFileName}.html"
                                    \$jsonReport = "\$executionFolder\\report_\${safeFileName}.json"
                                    \$resultadosFile = "\$executionFolder\\resultados.csv"
                                    
                                    Write-Host "========================================="
                                    Write-Host "Intentando ejecutar: \${folderName}"
                                    Write-Host "Ambiente: \${ambiente}"
                                    Write-Host "Carpeta: \${carpeta}"
                                    Write-Host "========================================="
                                    
                                    \$newmanExitCode = 0
                                    \$folderExists = \$false
                                    \$outputText = ""
                                    
                                    try {
                                        # Ejecutar Newman y capturar salida
                                        \$outputText = newman run "Collection/ACHDATA - YY.postman_collection.json" ``
                                            -e "Environment/ACHData QA.postman_environment.json" ``
                                            --folder "\${folderName}" ``
                                            --insecure ``
                                            --reporters cli,html,json ``
                                            --reporter-html-export "\${reportFile}" ``
                                            --reporter-json-export "\${jsonReport}" 2>&1 | Out-String
                                        
                                        \$newmanExitCode = \$LASTEXITCODE
                                        Write-Host \$outputText
                                        
                                        # Verificar si el error es por carpeta no encontrada
                                        if (\$outputText -match "Unable to find a folder") {
                                            Write-Host "[INFO] Carpeta no encontrada en la coleccion"
                                            \$folderExists = \$false
                                        } else {
                                            \$folderExists = \$true
                                        }
                                        
                                    } catch {
                                        Write-Host "Error ejecutando Newman: \$_"
                                        \$newmanExitCode = 1
                                        \$folderExists = \$false
                                    }
                                    
                                    # Procesar resultados
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
                                            Write-Host "[OK] Resultados para \${folderName}:"
                                            Write-Host "  Requests - Total: \${totalRequests} | Exitosos: \${passedRequests} | Fallidos: \${failedRequests}"
                                            Write-Host "  Assertions - Total: \${totalAssertions} | Exitosos: \${passedAssertions} | Fallidos: \${failedAssertions}"
                                            Write-Host ""
                                            
                                            # Guardar en CSV
                                            "\${ambiente},\${carpeta},EJECUTADO,\${totalRequests},\${passedRequests},\${failedRequests},\${totalAssertions},\${passedAssertions},\${failedAssertions}" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                        } catch {
                                            Write-Host "Error procesando JSON: \$_"
                                            "\${ambiente},\${carpeta},ERROR,0,0,0,0,0,0" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                        }
                                    } else {
                                        Write-Host "[INFO] Carpeta no disponible para este ambiente"
                                        "\${ambiente},\${carpeta},NO_DISPONIBLE,0,0,0,0,0,0" | Out-File -FilePath "\${resultadosFile}" -Append -Encoding UTF8
                                    }
                                    
                                    # Siempre retornar 0 para no fallar el pipeline
                                    exit 0
                                """)
                                
                                if (exitCode != 0) {
                                    echo "[WARN] Hubo un problema, pero continuamos con la ejecucion"
                                }
                            }
                        }
                    }
                }
            }
        }
        
        stage('Generar Resumen de Ejecucion') {
            steps {
                powershell """
                    \$executionFolder = "${env.EXECUTION_FOLDER}".Replace('/', '\\')
                    \$summaryFile = "\$executionFolder\\resumen_ejecucion.txt"
                    
                    "=========================================" | Out-File -FilePath \$summaryFile
                    "RESUMEN DE EJECUCION - ACHDATA" | Out-File -FilePath \$summaryFile -Append
                    "=========================================" | Out-File -FilePath \$summaryFile -Append
                    "Fecha: ${env.EXECUTION_DATE}" | Out-File -FilePath \$summaryFile -Append
                    "Hora: ${env.EXECUTION_TIME}" | Out-File -FilePath \$summaryFile -Append
                    "Ambiente: ${params.AMBIENTE}" | Out-File -FilePath \$summaryFile -Append
                    "Seccion: ${params.SECCION}" | Out-File -FilePath \$summaryFile -Append
                    "=========================================" | Out-File -FilePath \$summaryFile -Append
                    "" | Out-File -FilePath \$summaryFile -Append
                    
                    \$resultadosFile = "\$executionFolder\\resultados.csv"
                    
                    if (Test-Path "\$resultadosFile") {
                        "RESULTADOS DETALLADOS:" | Out-File -FilePath \$summaryFile -Append
                        "-----------------------" | Out-File -FilePath \$summaryFile -Append
                        "" | Out-File -FilePath \$summaryFile -Append
                        
                        \$lines = Get-Content "\$resultadosFile"
                        \$totalRequests = 0
                        \$totalPassedRequests = 0
                        \$totalFailedRequests = 0
                        \$totalAssertions = 0
                        \$totalPassedAssertions = 0
                        \$totalFailedAssertions = 0
                        \$ejecutados = 0
                        \$noDisponibles = 0
                        \$errores = 0
                        
                        foreach (\$line in \$lines) {
                            \$parts = \$line.Split(',')
                            if (\$parts.Count -eq 9) {
                                \$amb = \$parts[0].Trim()
                                \$carp = \$parts[1].Trim()
                                \$estado = \$parts[2].Trim()
                                \$reqTotal = [int]\$parts[3].Trim()
                                \$reqPassed = [int]\$parts[4].Trim()
                                \$reqFailed = [int]\$parts[5].Trim()
                                \$assTotal = [int]\$parts[6].Trim()
                                \$assPassed = [int]\$parts[7].Trim()
                                \$assFailed = [int]\$parts[8].Trim()
                                
                                # Determinar marcador segun estado
                                \$marcador = ""
                                if (\$estado -eq "EJECUTADO") {
                                    \$ejecutados++
                                    if (\$reqFailed -eq 0 -and \$assFailed -eq 0) {
                                        \$marcador = "[OK]"
                                    } else {
                                        \$marcador = "[WARN]"
                                    }
                                } elseif (\$estado -eq "NO_DISPONIBLE") {
                                    \$noDisponibles++
                                    \$marcador = "[N/A]"
                                } elseif (\$estado -eq "ERROR") {
                                    \$errores++
                                    \$marcador = "[ERROR]"
                                } else {
                                    \$marcador = "[?]"
                                }
                                
                                if (\$estado -eq "EJECUTADO") {
                                    \$totalRequests += \$reqTotal
                                    \$totalPassedRequests += \$reqPassed
                                    \$totalFailedRequests += \$reqFailed
                                    \$totalAssertions += \$assTotal
                                    \$totalPassedAssertions += \$assPassed
                                    \$totalFailedAssertions += \$assFailed
                                    
                                    \$formattedLine = "\$marcador \$amb - \$carp"
                                    \$formattedLine | Out-File -FilePath \$summaryFile -Append
                                    "   Requests: \$reqTotal (OK:\$reqPassed / FAIL:\$reqFailed) | Assertions: \$assTotal (OK:\$assPassed / FAIL:\$assFailed)" | Out-File -FilePath \$summaryFile -Append
                                } else {
                                    "\$marcador \$amb - \$carp [\$estado]" | Out-File -FilePath \$summaryFile -Append
                                }
                                "" | Out-File -FilePath \$summaryFile -Append
                            }
                        }
                        
                        "=========================================" | Out-File -FilePath \$summaryFile -Append
                        "RESUMEN GLOBAL:" | Out-File -FilePath \$summaryFile -Append
                        "-----------------------" | Out-File -FilePath \$summaryFile -Append
                        "Carpetas ejecutadas: \$ejecutados" | Out-File -FilePath \$summaryFile -Append
                        "Carpetas no disponibles: \$noDisponibles" | Out-File -FilePath \$summaryFile -Append
                        "Carpetas con error: \$errores" | Out-File -FilePath \$summaryFile -Append
                        "" | Out-File -FilePath \$summaryFile -Append
                        "TOTALES (solo ejecutadas):" | Out-File -FilePath \$summaryFile -Append
                        "Requests - Total: \$totalRequests | Exitosos: \$totalPassedRequests | Fallidos: \$totalFailedRequests" | Out-File -FilePath \$summaryFile -Append
                        "Assertions - Total: \$totalAssertions | Exitosos: \$totalPassedAssertions | Fallidos: \$totalFailedAssertions" | Out-File -FilePath \$summaryFile -Append
                        
                        if (\$totalRequests -gt 0) {
                            \$successRate = [math]::Round((\$totalPassedRequests / \$totalRequests) * 100, 2)
                            "Tasa de exito: \$successRate%" | Out-File -FilePath \$summaryFile -Append
                        }
                    } else {
                        "No se encontro el archivo de resultados" | Out-File -FilePath \$summaryFile -Append
                    }
                    
                    \$htmlReports = Get-ChildItem "\$executionFolder" -Filter "*.html"
                    "" | Out-File -FilePath \$summaryFile -Append
                    "=========================================" | Out-File -FilePath \$summaryFile -Append
                    "Total de reportes HTML generados: \$(\$htmlReports.Count)" | Out-File -FilePath \$summaryFile -Append
                    
                    Write-Host ""
                    Write-Host "=== RESUMEN DE EJECUCION ==="
                    Get-Content \$summaryFile
                """
            }
        }
        
        stage('Enviar Reportes por Correo') {
            steps {
                script {
                    def htmlFiles = powershell(returnStdout: true, script: """
                        \$executionFolder = "${env.EXECUTION_FOLDER}".Replace('/', '\\')
                        if (Test-Path "\$executionFolder") {
                            \$files = Get-ChildItem "\$executionFolder" -Filter "*.html" | Select-Object -ExpandProperty Name
                            if (\$files) {
                                Write-Output (\$files -join ",")
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
                        summaryContent = "No se pudo leer el resumen de ejecucion: ${e.message}"
                    }
                    
                    def emailBody = """
                    <html>
                    <body style="font-family: Arial, sans-serif;">
                        <h2>Reporte de Ejecucion Postman - ACHDATA</h2>
                        <p><strong>Fecha de ejecucion:</strong> ${env.EXECUTION_DATE} a las ${env.EXECUTION_TIME}</p>
                        <p><strong>Ambiente:</strong> ${params.AMBIENTE}</p>
                        <p><strong>Seccion:</strong> ${params.SECCION}</p>
                        
                        <h3>Resumen de Resultados:</h3>
                        <pre style="background-color: #f5f5f5; padding: 15px; border-radius: 5px; font-size: 12px; overflow-x: auto;">
${summaryContent}
                        </pre>
                        
                        <p><strong>Reportes adjuntos:</strong> ${htmlFiles != 'NO_FILES' && htmlFiles != 'NO_FOLDER' ? htmlFiles.split(',').size() : 0} archivos HTML</p>
                        
                        <hr>
                        <p style="font-size: 11px; color: #666;"><em>Este correo fue generado automaticamente por Jenkins.</em></p>
                    </body>
                    </html>
                    """
                    
                    echo "Enviando correo a: yeinerballesta@cbit-online.com"
                    
                    try {
                        emailext(
                            to: 'yeinerballesta@cbit-online.com',
                            subject: "Reporte Postman ACHDATA - ${params.AMBIENTE} - ${params.SECCION} - ${env.EXECUTION_DATE}",
                            body: emailBody,
                            mimeType: 'text/html',
                            attachmentsPattern: "${env.EXECUTION_FOLDER}/**/*.html",
                            from: 'jenkins@cbit-online.com'
                        )
                        echo "[OK] Correo enviado exitosamente"
                    } catch (Exception e) {
                        echo "[ERROR] No se pudo enviar el correo: ${e.message}"
                        echo "Verifica la configuracion de Email Extension Plugin en Jenkins"
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                echo "Proceso de ejecucion completado"
                echo "Reportes guardados en: ${env.EXECUTION_FOLDER}"
                
                try {
                    archiveArtifacts artifacts: "${env.EXECUTION_FOLDER}/**/*", allowEmptyArchive: true
                    echo "[OK] Artefactos archivados correctamente"
                } catch (Exception e) {
                    echo "[WARN] No se pudieron archivar los artefactos: ${e.message}"
                }
            }
        }
        success {
            echo '[OK] Pipeline completado exitosamente'
        }
        failure {
            echo '[ERROR] El pipeline fallo'
        }
    }
}