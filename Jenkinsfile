# Configuración inicial
$fecha = Get-Date -Format "yyyy-MM-dd"
$hora  = Get-Date -Format "HH-mm"
$timestamp = Get-Date -Format "yyyyMMdd_HHmm"
$baseReportPath = "Ejecuciones\ACHDATA\Fecha_$fecha`_Hora_$hora"
$basePath = "D:\Archivos"

# Archivos de datos
$fileIncluirExcluir = Join-Path $basePath "Incluir_Excluir Personas.csv"
$file50Registros = Join-Path $basePath "50 Registros.csv"
$fileCargueMasivo = Join-Path $basePath "cargue_masivo_usuarios.csv"

# Crear directorio de reportes
New-Item -ItemType Directory -Force -Path $baseReportPath | Out-Null

# Configuración de correo
$emailTo = "yeinerballesta@cbit-online.com"
$emailFrom = "yeinerballesta@cbit-online.com"
$smtpServer = "email.periferia-it.com"
$smtpPort = 587
$smtpUser = "yeinerballesta@cbit-online.com"
$smtpPassword = "Periferia2026*"

# Array para almacenar resultados
$resultados = @()

# Función para ejecutar colección y extraer resultados
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
    
    $safeFolderName = if ($folderName) { $folderName -replace '[\\/:*?"<>|]', '_' } else { "" }
    $reportSuffix = if ($safeFolderName) { "$collectionName`_$safeFolderName" } else { $collectionName }
    
    $reportHtml = "$baseReportPath\ACHData_$reportSuffix`_Report_$timestamp.html"
    $reportXml = "$baseReportPath\ACHData_$reportSuffix`_Report_$timestamp.xml"
    
    # Construir comando newman
    $newmanArgs = @(
        "run", $collectionFile,
        "--environment", "ACHData QA.postman_environment.json",
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
    
    # Parsear resultados de la salida
    $totalTests = 0
    $passed = 0
    $failed = 0
    
    # Buscar líneas con estadísticas
    foreach ($line in $output) {
        if ($line -match "assertions.*?(\d+)\s+failed.*?(\d+)\s+passed") {
            $failed = [int]$matches[1]
            $passed = [int]$matches[2]
            $totalTests = $failed + $passed
            break
        }
        elseif ($line -match "executed.*?(\d+)/(\d+)") {
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
    
    # Crear objeto PSCustomObject
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

# Ejecutar colección ACHDATA - YY con diferentes carpetas
# Ajusta los nombres de las carpetas según tu colección
$carpetas = @("Carpeta1", "Carpeta2", "Carpeta3")  # Modifica con los nombres reales de tus carpetas

foreach ($carpeta in $carpetas) {
    $resultados += Ejecutar-Coleccion -collectionName "YY" `
        -collectionFile "ACHDATA - YY.postman_collection.json" `
        -displayName "ACHDATA - YY - $carpeta" `
        -folderName $carpeta
}

# Calcular totales
$totalGeneral = ($resultados | Measure-Object -Property Total -Sum).Sum
$exitososGeneral = ($resultados | Measure-Object -Property Exitosos -Sum).Sum
$fallidosGeneral = ($resultados | Measure-Object -Property Fallidos -Sum).Sum

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
        <strong>Ubicacion reportes:</strong> $baseReportPath
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
    # Crear objeto de mensaje
    $mensaje = New-Object System.Net.Mail.MailMessage
    $mensaje.From = $emailFrom
    $mensaje.To.Add($emailTo)
    $mensaje.Subject = "Reporte Ejecucion APIs ACHData YY - $fecha $hora"
    $mensaje.Body = $cuerpoHtml
    $mensaje.IsBodyHtml = $true
    
    # Adjuntar solo reportes HTML
    foreach ($resultado in $resultados) {
        if (Test-Path $resultado.ReporteHtml) {
            $adjunto = New-Object System.Net.Mail.Attachment($resultado.ReporteHtml)
            $mensaje.Attachments.Add($adjunto)
        }
    }
    
    # Configurar cliente SMTP
    $smtp = New-Object System.Net.Mail.SmtpClient($smtpServer, $smtpPort)
    $smtp.EnableSsl = $true
    $smtp.Credentials = New-Object System.Net.NetworkCredential($smtpUser, $smtpPassword)
    
    # Enviar correo
    $smtp.Send($mensaje)
    
    Write-Host "Correo enviado exitosamente a: $emailTo" -ForegroundColor Green
    
    # Limpiar recursos
    $mensaje.Dispose()
    $smtp.Dispose()
}
catch {
    Write-Host "[ERROR] Error al enviar correo: $($_.Exception.Message)" -ForegroundColor Red
}

# Resumen final en consola
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

# Salir con código de error si hay fallos (para Jenkins)
if ($fallidosGeneral -gt 0) {
    exit 1
}