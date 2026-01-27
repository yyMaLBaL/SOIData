pipeline {
    agent any
    
    environment {
        EMAIL_RECIPIENT = 'yeinerballesta@cbit-online.com'
        SMTP_SERVER = 'email.periferia-it.com'
        SMTP_PORT = '587'
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
                    Write-Host "Verificando version de Node..." -ForegroundColor Cyan
                    node -v
                    Write-Host "Verificando version de Newman..." -ForegroundColor Cyan
                    newman -v
                '''
            }
        }
        
        stage('Preparar Directorio de Reportes') {
            steps {
                powershell '''
                    $fecha = Get-Date -Format "yyyy-MM-dd"
                    $hora = Get-Date -Format "HH-mm"
                    $timestamp = Get-Date -Format "yyyyMMdd_HHmm"
                    
                    $baseReportPath = "Ejecuciones\\ACHDATA\\Fecha_" + $fecha + "_Hora_" + $hora
                    
                    New-Item -ItemType Directory -Force -Path $baseReportPath | Out-Null
                    Write-Host "Directorio creado: $baseReportPath" -ForegroundColor Green
                    
                    # Guardar variables
                    $fecha | Out-File -FilePath "env_fecha.txt" -NoNewline -Encoding UTF8
                    $hora | Out-File -FilePath "env_hora.txt" -NoNewline -Encoding UTF8
                    $timestamp | Out-File -FilePath "env_timestamp.txt" -NoNewline -Encoding UTF8
                    $baseReportPath | Out-File -FilePath "env_reportpath.txt" -NoNewline -Encoding UTF8
                '''
            }
        }
        
        stage('Ejecutar Pruebas por Ambiente') {
            steps {
                powershell '''
                    # Cargar variables
                    $fecha = Get-Content "env_fecha.txt" -Encoding UTF8
                    $hora = Get-Content "env_hora.txt" -Encoding UTF8
                    $timestamp = Get-Content "env_timestamp.txt" -Encoding UTF8
                    $baseReportPath = Get-Content "env_reportpath.txt" -Encoding UTF8
                    
                    # Definir todos los ambientes a ejecutar
                    # Estructura: Ambiente / Carpetas que incluye
                    $ambientes = @(
                        @{
                            Nombre = "AT"
                            Carpetas = @(
                                "Autenticacion/AT - Autenticacion",
                                "Doble Factor de Autenticacion/AT - Doble Factor de Autenticacion",
                                "Uso de Datos/AT - Uso de Datos",
                                "Consultas/Detallada Natural Y Juridica/AT - Detallada Natural Y Juridica",
                                "Servicios ACH/AT - Servicios ACH",
                                "SDH/AT - SDH",
                                "Administracion/AT - Administracion",
                                "Monitoreo/AT - Monitoreo",
                                "Estadisticas/AT - Estadisticas",
                                "Auditoria/AT - Auditoria",
                                "ADMINISTRADOR ENTIDAD/AT - ADMINISTRADOR ENTIDAD"
                            )
                        },
                        @{
                            Nombre = "SS"
                            Carpetas = @(
                                "Autenticacion/SS - Autenticacion",
                                "Doble Factor de Autenticacion/SS - Doble Factor de Autenticacion",
                                "Uso de Datos/SS - Uso de Datos",
                                "Consultas/Detallada Natural Y Juridica/SS - Detallada Natural Y Juridica",
                                "Servicios ACH/SS - Servicios ACH",
                                "SDH/SS - SDH",
                                "Administracion/SS - Administracion",
                                "Monitoreo/SS - Monitoreo",
                                "Estadisticas/SS - Estadisticas",
                                "Auditoria/SS - Auditoria",
                                "ADMINISTRADOR ENTIDAD/SS - ADMINISTRADOR ENTIDAD"
                            )
                        },
                        @{
                            Nombre = "CS"
                            Carpetas = @(
                                "Autenticacion/CS - Autenticacion - API Consumo",
                                "Doble Factor de Autenticacion/CS - Doble Factor de Autenticacion - API Consumo",
                                "Uso de Datos/CS - Uso de Datos",
                                "Consultas/Detallada Natural Y Juridica/CS - Detallada Natural Y Juridica",
                                "Servicios ACH/CS - Servicios ACH - API Consumo",
                                "SDH/CS - SDH - API Consumo",
                                "Administracion/CS - Administracion - API Consumo",
                                "Monitoreo/CS - Monitoreo - API Consumo",
                                "Estadisticas/CS - Estadisticas - API Consumo",
                                "Auditoria/CS - Auditoria - API Consumo",
                                "ADMINISTRADOR ENTIDAD/CS - ADMINISTRADOR ENTIDAD - API Consumo"
                            )
                        },
                        @{
                            Nombre = "CER"
                            Carpetas = @(
                                "Autenticacion/CER - Autenticacion - API Security",
                                "Doble Factor de Autenticacion/CER - Doble Factor de Autenticacion - API Security",
                                "Uso de Datos/CER - Uso de Datos",
                                "Consultas/Detallada Natural Y Juridica/CER - Detallada Natural Y Juridica",
                                "Servicios ACH/CER - Servicios ACH - API Security",
                                "SDH/CER - SDH - API Security",
                                "Administracion/CER - Administracion - API Security",
                                "Monitoreo/CER - Monitoreo - API Security",
                                "Estadisticas/CER - Estadisticas - API Security",
                                "Auditoria/CER - Auditoria - API Security",
                                "ADMINISTRADOR ENTIDAD/CER - ADMINISTRADOR ENTIDAD - API Security"
                            )
                        },
                        @{
                            Nombre = "CERCS"
                            Carpetas = @(
                                "Autenticacion/CERCS - Autenticacion - API Security / API Consumo",
                                "Doble Factor de Autenticacion/CERCS - Doble Factor de Autenticacion - API Security / API Consumo",
                                "Uso de Datos/CERCS - Uso de Datos",
                                "Consultas/Detallada Natural Y Juridica/CERCS - Detallada Natural Y Juridica",
                                "Servicios ACH/CERCS - Servicios ACH - API Security / API Consumo",
                                "SDH/CERCS - SDH - API Security / API Consumo",
                                "Administracion/CERCS - Administracion - API Security / API Consumo",
                                "Monitoreo/CERCS - Monitoreo - API Security / API Consumo",
                                "Estadisticas/CERCS - Estadisticas - API Security / API Consumo",
                                "Auditoria/CERCS - Auditoria - API Security / API Consumo",
                                "ADMINISTRADOR ENTIDAD/CERCS - ADMINISTRADOR ENTIDAD - API Security / API Consumo"
                            )
                        }
                    )
                    
                    # Array para almacenar resultados
                    $resultados = @()
                    
                    foreach ($ambiente in $ambientes) {
                        $nombreAmbiente = $ambiente.Nombre
                        
                        Write-Host ""
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host "EJECUTANDO AMBIENTE: $nombreAmbiente" -ForegroundColor Cyan
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host ""
                        
                        $reportHTML = "$baseReportPath\\ACHData_" + $nombreAmbiente + "_Report_" + $timestamp + ".html"
                        $reportXML = "$baseReportPath\\ACHData_" + $nombreAmbiente + "_Report_" + $timestamp + ".xml"
                        
                        # Construir comando Newman con multiples folders
                        $foldersParam = ""
                        foreach ($carpeta in $ambiente.Carpetas) {
                            $foldersParam += "--folder `"$carpeta`" "
                        }
                        
                        $newmanCmd = "newman run `"Collection/ACHDATA - YY.postman_collection.json`" " +
                            "-e `"Environment/ACHData QA.postman_environment.json`" " +
                            $foldersParam +
                            "--env-var `"fileIncluirExcluir=File/Incluir_Excluir Personas.csv`" " +
                            "--env-var `"file50Registros=File/50 Registros.csv`" " +
                            "--env-var `"fileCargueMasivo=File/cargue_masivo_usuarios.csv`" " +
                            "--delay-request 1000 " +
                            "--timeout-request 31000 " +
                            "--reporters `"cli,htmlextra,junit`" " +
                            "--reporter-htmlextra-export `"$reportHTML`" " +
                            "--reporter-junit-export `"$reportXML`" " +
                            "--reporter-htmlextra-title `"ACHData $nombreAmbiente - $fecha $hora`" " +
                            "--reporter-htmlextra-showOnlyFails false " +
                            "--reporter-htmlextra-darkTheme " +
                            "--insecure"
                        
                        Write-Host "Ejecutando $($ambiente.Carpetas.Count) carpetas del ambiente $nombreAmbiente..." -ForegroundColor Yellow
                        
                        # Ejecutar Newman
                        Invoke-Expression $newmanCmd
                        $exitCode = $LASTEXITCODE
                        
                        # Procesar resultados del XML
                        if (Test-Path $reportXML) {
                            try {
                                [xml]$xmlContent = Get-Content $reportXML -Encoding UTF8
                                $testsuite = $xmlContent.testsuites.testsuite
                                
                                if ($testsuite) {
                                    $total = [int]$testsuite.tests
                                    $failures = [int]$testsuite.failures
                                    $errors = [int]$testsuite.errors
                                    $skipped = if ($testsuite.skipped) { [int]$testsuite.skipped } else { 0 }
                                    $exitosos = $total - $failures - $errors - $skipped
                                    $fallidos = $failures + $errors
                                } else {
                                    $total = 0
                                    $exitosos = 0
                                    $fallidos = 0
                                }
                                
                                $resultados += [PSCustomObject]@{
                                    Carpeta = $nombreAmbiente
                                    Total = $total
                                    Exitosos = $exitosos
                                    Fallidos = $fallidos
                                    ReporteHTML = $reportHTML
                                    ExitCode = $exitCode
                                }
                                
                                Write-Host ""
                                Write-Host "--- Resumen $nombreAmbiente ---" -ForegroundColor Yellow
                                Write-Host "Total: $total | Exitosos: $exitosos | Fallidos: $fallidos" -ForegroundColor Yellow
                                
                                if ($exitosos -eq $total -and $total -gt 0) {
                                    Write-Host "Todas las pruebas pasaron" -ForegroundColor Green
                                } elseif ($fallidos -gt 0) {
                                    Write-Host "Hay pruebas fallidas" -ForegroundColor Red
                                }
                            } catch {
                                Write-Host "Advertencia: No se pudo parsear XML para $nombreAmbiente - $_" -ForegroundColor Yellow
                                $resultados += [PSCustomObject]@{
                                    Carpeta = $nombreAmbiente
                                    Total = 0
                                    Exitosos = 0
                                    Fallidos = 0
                                    ReporteHTML = $reportHTML
                                    ExitCode = $exitCode
                                }
                            }
                        } else {
                            Write-Host "Advertencia: No se genero reporte XML para $nombreAmbiente" -ForegroundColor Yellow
                            $resultados += [PSCustomObject]@{
                                Carpeta = $nombreAmbiente
                                Total = 0
                                Exitosos = 0
                                Fallidos = 0
                                ReporteHTML = $reportHTML
                                ExitCode = $exitCode
                            }
                        }
                        
                        Write-Host ""
                        Write-Host "Esperando 3 segundos antes del siguiente ambiente..." -ForegroundColor Gray
                        Start-Sleep -Seconds 3
                    }
                    
                    # Guardar resultados
                    $resultados | ConvertTo-Json -Depth 10 | Out-File "resultados.json" -Encoding UTF8
                    
                    # Resumen global
                    Write-Host ""
                    Write-Host "========================================" -ForegroundColor Green
                    Write-Host "RESUMEN GLOBAL DE EJECUCION" -ForegroundColor Green
                    Write-Host "========================================" -ForegroundColor Green
                    
                    if ($resultados.Count -gt 0) {
                        $totalGlobal = ($resultados | Measure-Object -Property Total -Sum).Sum
                        $exitososGlobal = ($resultados | Measure-Object -Property Exitosos -Sum).Sum
                        $fallidosGlobal = ($resultados | Measure-Object -Property Fallidos -Sum).Sum
                        
                        Write-Host "Ambientes ejecutados: $($resultados.Count)" -ForegroundColor Cyan
                        Write-Host "Total de pruebas: $totalGlobal" -ForegroundColor Cyan
                        Write-Host "Exitosas: $exitososGlobal" -ForegroundColor Green
                        Write-Host "Fallidas: $fallidosGlobal" -ForegroundColor Red
                        Write-Host "Reportes guardados en: $baseReportPath" -ForegroundColor Cyan
                    } else {
                        Write-Host "No se ejecutaron pruebas" -ForegroundColor Red
                    }
                '''
            }
        }
        
        stage('Enviar Reportes por Email') {
            steps {
                powershell '''
                    # Cargar variables
                    $fecha = Get-Content "env_fecha.txt" -Encoding UTF8
                    $hora = Get-Content "env_hora.txt" -Encoding UTF8
                    $baseReportPath = Get-Content "env_reportpath.txt" -Encoding UTF8
                    
                    # Cargar resultados
                    if (-not (Test-Path "resultados.json")) {
                        Write-Host "No hay resultados para enviar" -ForegroundColor Yellow
                        exit 0
                    }
                    
                    $resultadosJson = Get-Content "resultados.json" -Encoding UTF8 | ConvertFrom-Json
                    
                    if ($resultadosJson.Count -eq 0) {
                        Write-Host "No hay resultados para enviar" -ForegroundColor Yellow
                        exit 0
                    }
                    
                    # Calcular totales
                    $totalGlobal = ($resultadosJson | Measure-Object -Property Total -Sum).Sum
                    $exitososGlobal = ($resultadosJson | Measure-Object -Property Exitosos -Sum).Sum
                    $fallidosGlobal = ($resultadosJson | Measure-Object -Property Fallidos -Sum).Sum
                    
                    # Determinar estado general
                    if ($fallidosGlobal -eq 0 -and $totalGlobal -gt 0) {
                        $estadoGeneral = "EXITOSA"
                        $colorEstado = "#27ae60"
                    } elseif ($fallidosGlobal -gt 0) {
                        $estadoGeneral = "CON FALLOS"
                        $colorEstado = "#e74c3c"
                    } else {
                        $estadoGeneral = "SIN PRUEBAS"
                        $colorEstado = "#95a5a6"
                    }
                    
                    # Construir cuerpo del email
                    $cuerpoHTML = @"
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <style>
        body { 
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; 
            background-color: #f4f4f4;
            margin: 0;
            padding: 20px;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
            background-color: white;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
            overflow: hidden;
        }
        .header { 
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white; 
            padding: 30px;
            text-align: center;
        }
        .header h1 {
            margin: 0;
            font-size: 28px;
        }
        .header p {
            margin: 10px 0 0 0;
            opacity: 0.9;
        }
        .content {
            padding: 30px;
        }
        .status-badge {
            display: inline-block;
            padding: 8px 16px;
            border-radius: 20px;
            font-weight: bold;
            margin: 10px 0;
            background-color: $colorEstado;
            color: white;
        }
        .summary-box {
            background-color: #f8f9fa;
            border-left: 4px solid #667eea;
            padding: 20px;
            margin: 20px 0;
            border-radius: 4px;
        }
        .summary-box h3 {
            margin-top: 0;
            color: #2c3e50;
        }
        .stats {
            display: flex;
            justify-content: space-around;
            margin: 20px 0;
        }
        .stat-item {
            text-align: center;
            padding: 15px;
        }
        .stat-number {
            font-size: 32px;
            font-weight: bold;
            display: block;
        }
        .stat-label {
            color: #7f8c8d;
            font-size: 14px;
            text-transform: uppercase;
        }
        table { 
            border-collapse: collapse; 
            width: 100%; 
            margin: 20px 0;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        }
        th { 
            background-color: #34495e;
            color: white; 
            padding: 14px;
            text-align: left;
            font-weight: 600;
        }
        td { 
            border: 1px solid #ddd; 
            padding: 12px;
            background-color: white;
        }
        tr:hover td {
            background-color: #f8f9fa;
        }
        .exitoso { 
            color: #27ae60; 
            font-weight: bold;
        }
        .fallido { 
            color: #e74c3c; 
            font-weight: bold;
        }
        .total {
            color: #3498db;
            font-weight: bold;
        }
        .footer { 
            background-color: #ecf0f1;
            padding: 20px;
            text-align: center;
            font-size: 12px; 
            color: #7f8c8d;
            border-top: 1px solid #ddd;
        }
        .footer p {
            margin: 5px 0;
        }
        .info-box {
            background-color: #e3f2fd;
            border-left: 4px solid #2196f3;
            padding: 15px;
            margin: 20px 0;
            border-radius: 4px;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>Resultados de Pruebas - ACHData</h1>
            <p>Fecha: $fecha | Hora: $hora</p>
            <div class="status-badge">$estadoGeneral</div>
        </div>
        
        <div class="content">
            <div class="summary-box">
                <h3>Resumen Global - Todos los Ambientes</h3>
                <div class="stats">
                    <div class="stat-item">
                        <span class="stat-number total">$totalGlobal</span>
                        <span class="stat-label">Total</span>
                    </div>
                    <div class="stat-item">
                        <span class="stat-number exitoso">$exitososGlobal</span>
                        <span class="stat-label">Exitosos</span>
                    </div>
                    <div class="stat-item">
                        <span class="stat-number fallido">$fallidosGlobal</span>
                        <span class="stat-label">Fallidos</span>
                    </div>
                </div>
            </div>
            
            <h3>Detalle por Ambiente</h3>
            <table>
                <thead>
                    <tr>
                        <th>Ambiente</th>
                        <th style="text-align: center;">Total Casos</th>
                        <th style="text-align: center;">Exitosos</th>
                        <th style="text-align: center;">Fallidos</th>
                        <th style="text-align: center;">Estado</th>
                    </tr>
                </thead>
                <tbody>
"@
                    
                    foreach ($resultado in $resultadosJson) {
                        if ($resultado.Fallidos -eq 0 -and $resultado.Total -gt 0) {
                            $estadoCarpeta = "PASS"
                            $colorEstadoCarpeta = "exitoso"
                        } elseif ($resultado.Fallidos -gt 0) {
                            $estadoCarpeta = "FAIL"
                            $colorEstadoCarpeta = "fallido"
                        } else {
                            $estadoCarpeta = "N/A"
                            $colorEstadoCarpeta = ""
                        }
                        
                        $cuerpoHTML += @"
                    <tr>
                        <td><strong>$($resultado.Carpeta)</strong></td>
                        <td class="total" style="text-align: center;">$($resultado.Total)</td>
                        <td class="exitoso" style="text-align: center;">$($resultado.Exitosos)</td>
                        <td class="fallido" style="text-align: center;">$($resultado.Fallidos)</td>
                        <td class="$colorEstadoCarpeta" style="text-align: center;">$estadoCarpeta</td>
                    </tr>
"@
                    }
                    
                    $cuerpoHTML += @"
                </tbody>
            </table>
            
            <div class="info-box">
                <strong>Archivos Adjuntos:</strong>
                <p>Se adjuntan 5 reportes HTML detallados (uno por ambiente: AT, SS, CS, CER, CERCS). Cada reporte contiene todas las carpetas ejecutadas para ese ambiente.</p>
            </div>
            
            <div class="info-box">
                <strong>Ubicacion de Reportes:</strong>
                <p><code>$baseReportPath</code></p>
            </div>
        </div>
        
        <div class="footer">
            <p><strong>Generado automaticamente por Jenkins</strong></p>
            <p>Pipeline: <strong>$env:JOB_NAME</strong> | Build: <strong>#$env:BUILD_NUMBER</strong></p>
            <p>Servidor: <strong>$env:COMPUTERNAME</strong></p>
            <p style="margin-top: 10px;">ACHData - Sistema de Pruebas Automatizadas</p>
        </div>
    </div>
</body>
</html>
"@
                    
                    # Recopilar reportes HTML
                    $reportesHTML = @()
                    foreach ($resultado in $resultadosJson) {
                        if (Test-Path $resultado.ReporteHTML) {
                            $reportesHTML += $resultado.ReporteHTML
                        }
                    }
                    
                    Write-Host ""
                    Write-Host "========================================" -ForegroundColor Cyan
                    Write-Host "PREPARANDO ENVIO DE EMAIL" -ForegroundColor Cyan
                    Write-Host "========================================" -ForegroundColor Cyan
                    Write-Host "Reportes HTML encontrados: $($reportesHTML.Count)" -ForegroundColor Cyan
                    
                    # Configurar email
                    $From = "jenkins@periferia-it.com"
                    $To = $env:EMAIL_RECIPIENT
                    $Subject = "ACHData - Resultados de Pruebas [$estadoGeneral] - $fecha $hora"
                    
                    try {
                        # Crear mensaje
                        $Message = New-Object System.Net.Mail.MailMessage
                        $Message.From = $From
                        $Message.To.Add($To)
                        $Message.Subject = $Subject
                        $Message.Body = $cuerpoHTML
                        $Message.IsBodyHtml = $true
                        $Message.Priority = [System.Net.Mail.MailPriority]::Normal
                        
                        # Adjuntar reportes HTML
                        foreach ($reporte in $reportesHTML) {
                            if (Test-Path $reporte) {
                                $attachment = New-Object System.Net.Mail.Attachment($reporte)
                                $Message.Attachments.Add($attachment)
                                $nombreArchivo = Split-Path $reporte -Leaf
                                Write-Host "Adjuntado: $nombreArchivo" -ForegroundColor Green
                            }
                        }
                        
                        # Configurar SMTP
                        $SMTPClient = New-Object System.Net.Mail.SmtpClient($env:SMTP_SERVER, $env:SMTP_PORT)
                        $SMTPClient.EnableSsl = $true
                        
                        # Usar credenciales de Jenkins si estan disponibles
                        if ($env:EMAIL_USER -and $env:EMAIL_PASSWORD) {
                            $SMTPClient.Credentials = New-Object System.Net.NetworkCredential(
                                $env:EMAIL_USER, 
                                $env:EMAIL_PASSWORD
                            )
                            Write-Host "Usando credenciales configuradas" -ForegroundColor Yellow
                        } else {
                            Write-Host "ADVERTENCIA: No hay credenciales configuradas. El envio puede fallar." -ForegroundColor Yellow
                        }
                        
                        # Enviar email
                        Write-Host ""
                        Write-Host "Enviando email a: $To" -ForegroundColor Cyan
                        Write-Host "Servidor SMTP: $($env:SMTP_SERVER):$($env:SMTP_PORT)" -ForegroundColor Cyan
                        $SMTPClient.Send($Message)
                        Write-Host "Email enviado exitosamente" -ForegroundColor Green
                        
                    } catch {
                        Write-Host ""
                        Write-Host "Error al enviar email" -ForegroundColor Red
                        Write-Host "Detalles: $($_.Exception.Message)" -ForegroundColor Red
                        
                        if ($_.Exception.InnerException) {
                            Write-Host "Error interno: $($_.Exception.InnerException.Message)" -ForegroundColor Red
                        }
                        
                        Write-Host ""
                        Write-Host "Los reportes estan disponibles en: $baseReportPath" -ForegroundColor Yellow
                        Write-Host "Continuando sin envio de email..." -ForegroundColor Yellow
                        
                    } finally {
                        # Limpiar recursos
                        if ($Message) { $Message.Dispose() }
                        if ($SMTPClient) { $SMTPClient.Dispose() }
                    }
                '''
            }
        }
    }
    
    post {
        always {
            powershell '''
                Write-Host ""
                Write-Host "========================================" -ForegroundColor Magenta
                Write-Host "LIMPIEZA DE ARCHIVOS TEMPORALES" -ForegroundColor Magenta
                Write-Host "========================================" -ForegroundColor Magenta
                
                # Limpiar archivos temporales
                $tempFiles = @("env_fecha.txt", "env_hora.txt", "env_timestamp.txt", "env_reportpath.txt", "resultados.json")
                
                foreach ($file in $tempFiles) {
                    if (Test-Path $file) {
                        Remove-Item $file -Force
                        Write-Host "Eliminado: $file" -ForegroundColor Gray
                    }
                }
                
                Write-Host "Limpieza completada" -ForegroundColor Green
                Write-Host ""
            '''
        }
        failure {
            echo 'Fallaron las pruebas Postman'
        }
        success {
            echo 'Ejecucion completada correctamente y reportes enviados'
        }
    }
}