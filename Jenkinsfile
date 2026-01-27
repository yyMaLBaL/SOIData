pipeline {
    agent any
    
    environment {
        BASE_PATH = 'D:\\Archivos'
        REPORT_PATH = 'D:\\Ejecuciones'
        ENV_FILE = 'ACHData QA.postman_environment.json'
        COLLECTION_FILE = 'ACHDATA - YY.postman_collection.json'
        EMAIL_TO = 'yeinerballesta@cbit-online.com'
        EMAIL_FROM = 'yeinerballesta@cbit-online.com'
        SMTP_SERVER = 'email.periferia-it.com'
        SMTP_PORT = '587'
        SMTP_USER = 'yeinerballesta@cbit-online.com'
        SMTP_PASSWORD = 'Periferia2026*'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/yyMaLBaL/SOIData.git',
                    branch: 'main'
            }
        }
        
        stage('Preparar Entorno') {
            steps {
                script {
                    powershell '''
                        Write-Host "Verificando archivos necesarios..." -ForegroundColor Cyan
                        
                        # Verificar archivos de datos
                        $requiredFiles = @(
                            "$env:BASE_PATH\\Incluir_Excluir Personas.csv",
                            "$env:BASE_PATH\\50 Registros.csv",
                            "$env:BASE_PATH\\cargue_masivo_usuarios.csv"
                        )
                        
                        foreach ($file in $requiredFiles) {
                            if (Test-Path $file) {
                                Write-Host "  [OK] $file" -ForegroundColor Green
                            } else {
                                Write-Host "  [FALTA] $file" -ForegroundColor Red
                                throw "Archivo requerido no encontrado: $file"
                            }
                        }
                        
                        # Crear directorio de reportes
                        if (-not (Test-Path $env:REPORT_PATH)) {
                            New-Item -ItemType Directory -Force -Path $env:REPORT_PATH | Out-Null
                            Write-Host "Directorio de reportes creado: $env:REPORT_PATH" -ForegroundColor Green
                        }
                        
                        Write-Host "Entorno preparado correctamente" -ForegroundColor Green
                    '''
                }
            }
        }
        
        stage('Ejecutar Colecciones Newman') {
            steps {
                script {
                    powershell '''
                        # Configuración inicial
                        $fecha = Get-Date -Format "yyyy-MM-dd"
                        $hora  = Get-Date -Format "HH-mm"
                        $timestamp = Get-Date -Format "yyyyMMdd_HHmm"
                        $baseReportPath = "$env:REPORT_PATH\\ACHDATA\\Fecha_$fecha`_Hora_$hora"
                        $basePath = $env:BASE_PATH
                        
                        # Archivos de datos
                        $fileIncluirExcluir = "$basePath\\Incluir_Excluir Personas.csv"
                        $file50Registros = "$basePath\\50 Registros.csv"
                        $fileCargueMasivo = "$basePath\\cargue_masivo_usuarios.csv"
                        
                        # Crear directorio de reportes
                        New-Item -ItemType Directory -Force -Path $baseReportPath | Out-Null
                        
                        # Array para almacenar resultados
                        $resultados = @()
                        
                        # Función para ejecutar colección
                        function Ejecutar-Coleccion {
                            param(
                                [string]$collectionName,
                                [string]$collectionFile,
                                [string]$displayName,
                                [string]$folderName = $null
                            )
                            
                            Write-Host ""
                            Write-Host "========================================" -ForegroundColor Cyan
                            Write-Host "Iniciando ejecucion: $displayName" -ForegroundColor Cyan
                            if ($folderName) {
                                Write-Host "Carpeta: $folderName" -ForegroundColor Cyan
                            }
                            Write-Host "========================================" -ForegroundColor Cyan
                            Write-Host ""
                            
                            $safeFolderName = if ($folderName) { $folderName -replace '[\\\\/:*?"<>|]', '_' } else { "" }
                            $reportSuffix = if ($safeFolderName) { "$collectionName`_$safeFolderName" } else { $collectionName }
                            
                            $reportHtml = "$baseReportPath\\ACHData_$reportSuffix`_Report_$timestamp.html"
                            $reportXml = "$baseReportPath\\ACHData_$reportSuffix`_Report_$timestamp.xml"
                            
                            # Construir comando newman
                            $newmanArgs = @(
                                "run", $collectionFile,
                                "--environment", $env:ENV_FILE,
                                "--env-var", "fileIncluirExcluir=$fileIncluirExcluir",
                                "--env-var", "file50Registros=$file50Registros",
                                "--env-var", "fileCargueMasivo=$fileCargueMasivo",
                                "--delay-request", "1000",
                                "--iteration-count", "1",
                                "--timeout-request", "31000",
                                "--reporters", "cli,htmlextra,junit",
                                "--reporter-htmlextra-export", $reportHtml,
                                "--reporter-junit-export", $reportXml,
                                "--reporter-htmlextra-title", "ACHData $collectionName - $fecha $hora",
                                "--reporter-htmlextra-logs",
                                "--reporter-htmlextra-showOnlyFails",
                                "--insecure"
                            )
                            
                            # Agregar folder si se especifica
                            if ($folderName) {
                                $newmanArgs += "--folder"
                                $newmanArgs += $folderName
                            }
                            
                            # Ejecutar newman
                            $output = & newman $newmanArgs 2>&1 | Tee-Object -Variable output
                            
                            # Parsear resultados
                            $totalTests = 0
                            $passed = 0
                            $failed = 0
                            
                            foreach ($line in $output) {
                                if ($line -match "assertions.*?(\\d+)\\s+failed.*?(\\d+)\\s+passed") {
                                    $failed = [int]$matches[1]
                                    $passed = [int]$matches[2]
                                    $totalTests = $failed + $passed
                                    break
                                }
                                elseif ($line -match "executed.*?(\\d+)/(\\d+)") {
                                    $passed = [int]$matches[1]
                                    $totalTests = [int]$matches[2]
                                    $failed = $totalTests - $passed
                                    break
                                }
                            }
                            
                            # Si no se pudo parsear, intentar leer del XML
                            if ($totalTests -eq 0 -and (Test-Path $reportXml)) {
                                try {
                                    [xml]$xmlContent = Get-Content $reportXml
                                    $testsuites = $xmlContent.testsuites
                                    if ($testsuites) {
                                        $totalTests = [int]$testsuites.tests
                                        $failed = [int]$testsuites.failures + [int]$testsuites.errors
                                        $passed = $totalTests - $failed
                                    }
                                }
                                catch {
                                    Write-Host "No se pudo parsear el archivo XML" -ForegroundColor Yellow
                                }
                            }
                            
                            # Crear objeto resultado
                            $resultado = [PSCustomObject]@{
                                Coleccion = $displayName
                                Total = $totalTests
                                Exitosos = $passed
                                Fallidos = $failed
                                ReporteHtml = $reportHtml
                                ExitCode = $LASTEXITCODE
                            }
                            
                            if ($LASTEXITCODE -eq 0) {
                                Write-Host ""
                                Write-Host "Ejecucion completada exitosamente" -ForegroundColor Green
                            } else {
                                Write-Host ""
                                Write-Host "Ejecucion completada con errores" -ForegroundColor Red
                            }
                            
                            Write-Host "Reporte HTML: $reportHtml" -ForegroundColor Cyan
                            Write-Host "Tests ejecutados: $totalTests | Exitosos: $passed | Fallidos: $failed" -ForegroundColor White
                            Write-Host ""
                            
                            return $resultado
                        }
                        
                        # Definir carpetas a ejecutar (ajusta según tu colección)
                        $carpetas = @("Carpeta1", "Carpeta2", "Carpeta3")
                        
                        # Ejecutar cada carpeta
                        foreach ($carpeta in $carpetas) {
                            $resultados += Ejecutar-Coleccion -collectionName "YY" `
                                -collectionFile $env:COLLECTION_FILE `
                                -displayName "ACHDATA - YY - $carpeta" `
                                -folderName $carpeta
                        }
                        
                        # Calcular totales
                        $totalGeneral = ($resultados | Measure-Object -Property Total -Sum).Sum
                        $exitososGeneral = ($resultados | Measure-Object -Property Exitosos -Sum).Sum
                        $fallidosGeneral = ($resultados | Measure-Object -Property Fallidos -Sum).Sum
                        
                        # Guardar resultados en variable de entorno para uso posterior
                        [Environment]::SetEnvironmentVariable("TOTAL_TESTS", $totalGeneral, "Process")
                        [Environment]::SetEnvironmentVariable("PASSED_TESTS", $exitososGeneral, "Process")
                        [Environment]::SetEnvironmentVariable("FAILED_TESTS", $fallidosGeneral, "Process")
                        [Environment]::SetEnvironmentVariable("REPORT_PATH_FINAL", $baseReportPath, "Process")
                        
                        # Serializar resultados para el correo
                        $resultadosJson = $resultados | ConvertTo-Json -Compress
                        [Environment]::SetEnvironmentVariable("RESULTADOS_JSON", $resultadosJson, "Process")
                        
                        # Resumen final
                        Write-Host ""
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host "RESUMEN FINAL" -ForegroundColor Cyan
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host "Total de casos ejecutados: $totalGeneral" -ForegroundColor White
                        Write-Host "Casos exitosos: $exitososGeneral" -ForegroundColor Green
                        Write-Host "Casos fallidos: $fallidosGeneral" -ForegroundColor Red
                        Write-Host "Reportes guardados en: $baseReportPath" -ForegroundColor Cyan
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host ""
                        
                        # Salir con error si hay fallos
                        if ($fallidosGeneral -gt 0) {
                            exit 1
                        }
                    '''
                }
            }
        }
        
        stage('Enviar Reporte') {
            steps {
                script {
                    powershell '''
                        $fecha = Get-Date -Format "yyyy-MM-dd"
                        $hora  = Get-Date -Format "HH-mm"
                        
                        # Recuperar variables de entorno
                        $totalGeneral = [int]$env:TOTAL_TESTS
                        $exitososGeneral = [int]$env:PASSED_TESTS
                        $fallidosGeneral = [int]$env:FAILED_TESTS
                        $baseReportPath = $env:REPORT_PATH_FINAL
                        
                        # Deserializar resultados
                        $resultados = $env:RESULTADOS_JSON | ConvertFrom-Json
                        
                        # Preparar cuerpo del correo
                        $cuerpoHtml = @"
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        h1 { color: #0a0a0a; }
        h2 { color: #34495e; margin-top: 30px; }
        table { border-collapse: collapse; width: 100%; margin: 20px 0; }
        th, td { border: 1px solid #ddd; padding: 12px; text-align: left; }
        th { background-color: #0b6730; color: white; }
        tr:nth-child(even) { background-color: #f2f2f2; }
        .exito { color: #308446; font-weight: bold; }
        .fallo { color: #cc0605; font-weight: bold; }
        .info { background-color: #ecf0f1; padding: 15px; border-radius: 5px; margin: 20px 0; }
    </style>
</head>
<body>
    <h1>Reporte de Ejecucion - APIs ACHData YY</h1>
    <div class="info">
        <strong>Fecha:</strong> $fecha<br>
        <strong>Hora:</strong> $hora<br>
        <strong>Ubicacion reportes:</strong> $baseReportPath<br>
        <strong>Jenkins Job:</strong> $env:JOB_NAME<br>
        <strong>Build Number:</strong> #$env:BUILD_NUMBER
    </div>
    
    <h2>Resumen de Resultados</h2>
    <table>
        <tr>
            <th>Coleccion</th>
            <th>Total Casos</th>
            <th>Exitosos</th>
            <th>Fallidos</th>
            <th>Estado</th>
        </tr>
"@
                        
                        foreach ($resultado in $resultados) {
                            $estado = if ($resultado.Fallidos -eq 0) { 
                                "<span class='exito'>Exitoso</span>" 
                            } else { 
                                "<span class='fallo'>ERROR - Con Fallos</span>" 
                            }
                            
                            $cuerpoHtml += @"
        <tr>
            <td><strong>$($resultado.Coleccion)</strong></td>
            <td>$($resultado.Total)</td>
            <td class='exito'>$($resultado.Exitosos)</td>
            <td class='fallo'>$($resultado.Fallidos)</td>
            <td>$estado</td>
        </tr>
"@
                        }
                        
                        $cuerpoHtml += @"
        <tr style='background-color: #d5dbdb; font-weight: bold;'>
            <td>TOTAL</td>
            <td>$totalGeneral</td>
            <td class='exito'>$exitososGeneral</td>
            <td class='fallo'>$fallidosGeneral</td>
            <td></td>
        </tr>
    </table>
    
    <p style='margin-top: 30px; color: #7f8c8d;'>
        <em>Los reportes HTML detallados estan adjuntos a este correo.</em>
    </p>
</body>
</html>
"@
                        
                        # Enviar correo
                        Write-Host ""
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host "Enviando reporte por correo electronico..." -ForegroundColor Cyan
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host ""
                        
                        try {
                            $mensaje = New-Object System.Net.Mail.MailMessage
                            $mensaje.From = $env:EMAIL_FROM
                            $mensaje.To.Add($env:EMAIL_TO)
                            $mensaje.Subject = "Reporte Ejecucion APIs ACHData YY - $fecha $hora - Build #$env:BUILD_NUMBER"
                            $mensaje.Body = $cuerpoHtml
                            $mensaje.IsBodyHtml = $true
                            
                            # Adjuntar reportes HTML
                            foreach ($resultado in $resultados) {
                                if (Test-Path $resultado.ReporteHtml) {
                                    $adjunto = New-Object System.Net.Mail.Attachment($resultado.ReporteHtml)
                                    $mensaje.Attachments.Add($adjunto)
                                }
                            }
                            
                            # Configurar SMTP
                            $smtp = New-Object System.Net.Mail.SmtpClient($env:SMTP_SERVER, $env:SMTP_PORT)
                            $smtp.EnableSsl = $true
                            $smtp.Credentials = New-Object System.Net.NetworkCredential($env:SMTP_USER, $env:SMTP_PASSWORD)
                            
                            # Enviar
                            $smtp.Send($mensaje)
                            
                            Write-Host "Correo enviado exitosamente a: $env:EMAIL_TO" -ForegroundColor Green
                            
                            # Limpiar
                            $mensaje.Dispose()
                            $smtp.Dispose()
                        }
                        catch {
                            Write-Host "[ERROR] Error al enviar correo: $($_.Exception.Message)" -ForegroundColor Red
                        }
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo 'Todas las colecciones se ejecutaron correctamente'
        }
        failure {
            echo 'Una o más colecciones fallaron. Revisa el reporte de correo.'
        }
        always {
            script {
                powershell '''
                    Write-Host "Pipeline finalizado" -ForegroundColor Cyan
                    Write-Host "Total Tests: $env:TOTAL_TESTS" -ForegroundColor White
                    Write-Host "Passed: $env:PASSED_TESTS" -ForegroundColor Green
                    Write-Host "Failed: $env:FAILED_TESTS" -ForegroundColor Red
                '''
            }
        }
    }
}