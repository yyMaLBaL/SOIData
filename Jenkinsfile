pipeline {
    agent any
    
    environment {
        // Rutas de archivos
        ARCHIVOS_PATH = 'D:\\ACHData\\SOIData\\Archivos'
        COLECCIONES_PATH = 'D:\\ACHData\\SOIData\\Colecciones'
        ENTORNOS_PATH = 'D:\\ACHData\\SOIData\\Entornos'
        REPORT_PATH = 'D:\\Ejecuciones'
        
        // Archivos específicos
        ENV_FILE = "${ENTORNOS_PATH}\\ACHData QA.postman_environment.json"
        COLLECTION_FILE = "${COLECCIONES_PATH}\\ACHDATA - YY.postman_collection.json"
        
        // Configuración de correo
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
                        Write-Host ""
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host "VERIFICANDO ENTORNO" -ForegroundColor Cyan
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host ""
                        
                        # Verificar directorios principales
                        Write-Host "Verificando directorios..." -ForegroundColor Yellow
                        
                        $directories = @{
                            "Archivos" = $env:ARCHIVOS_PATH
                            "Colecciones" = $env:COLECCIONES_PATH
                            "Entornos" = $env:ENTORNOS_PATH
                            "Reportes" = $env:REPORT_PATH
                        }
                        
                        foreach ($dir in $directories.GetEnumerator()) {
                            if (Test-Path $dir.Value) {
                                Write-Host "  [OK] $($dir.Key): $($dir.Value)" -ForegroundColor Green
                            } else {
                                Write-Host "  [CREANDO] $($dir.Key): $($dir.Value)" -ForegroundColor Yellow
                                New-Item -ItemType Directory -Force -Path $dir.Value | Out-Null
                                Write-Host "  [OK] $($dir.Key) creado" -ForegroundColor Green
                            }
                        }
                        
                        Write-Host ""
                        Write-Host "Verificando archivos de datos..." -ForegroundColor Yellow
                        
                        # Verificar archivos CSV
                        $csvFiles = @(
                            "$env:ARCHIVOS_PATH\\Incluir_Excluir Personas.csv",
                            "$env:ARCHIVOS_PATH\\50 Registros.csv",
                            "$env:ARCHIVOS_PATH\\cargue_masivo_usuarios.csv"
                        )
                        
                        $allCsvExist = $true
                        foreach ($file in $csvFiles) {
                            if (Test-Path $file) {
                                Write-Host "  [OK] $(Split-Path $file -Leaf)" -ForegroundColor Green
                            } else {
                                Write-Host "  [FALTA] $(Split-Path $file -Leaf)" -ForegroundColor Red
                                $allCsvExist = $false
                            }
                        }
                        
                        Write-Host ""
                        Write-Host "Verificando colección Postman..." -ForegroundColor Yellow
                        
                        if (Test-Path $env:COLLECTION_FILE) {
                            Write-Host "  [OK] ACHDATA - YY.postman_collection.json" -ForegroundColor Green
                        } else {
                            Write-Host "  [ERROR] No se encontro: $env:COLLECTION_FILE" -ForegroundColor Red
                            throw "Archivo de coleccion no encontrado"
                        }
                        
                        Write-Host ""
                        Write-Host "Verificando entorno Postman..." -ForegroundColor Yellow
                        
                        if (Test-Path $env:ENV_FILE) {
                            Write-Host "  [OK] ACHData QA.postman_environment.json" -ForegroundColor Green
                        } else {
                            Write-Host "  [ERROR] No se encontro: $env:ENV_FILE" -ForegroundColor Red
                            throw "Archivo de entorno no encontrado"
                        }
                        
                        Write-Host ""
                        if (-not $allCsvExist) {
                            Write-Host "ADVERTENCIA: Algunos archivos CSV no existen." -ForegroundColor Yellow
                            Write-Host "Las pruebas que dependan de estos archivos pueden fallar." -ForegroundColor Yellow
                        } else {
                            Write-Host "Todos los archivos verificados correctamente" -ForegroundColor Green
                        }
                        
                        Write-Host ""
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host "ENTORNO PREPARADO" -ForegroundColor Cyan
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host ""
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
                        
                        # Archivos de datos
                        $fileIncluirExcluir = "$env:ARCHIVOS_PATH\\Incluir_Excluir Personas.csv"
                        $file50Registros = "$env:ARCHIVOS_PATH\\50 Registros.csv"
                        $fileCargueMasivo = "$env:ARCHIVOS_PATH\\cargue_masivo_usuarios.csv"
                        
                        Write-Host ""
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host "CONFIGURACION DE EJECUCION" -ForegroundColor Cyan
                        Write-Host "========================================" -ForegroundColor Cyan
                        Write-Host "Coleccion: $env:COLLECTION_FILE" -ForegroundColor White
                        Write-Host "Entorno: $env:ENV_FILE" -ForegroundColor White
                        Write-Host "Reportes: $baseReportPath" -ForegroundColor White
                        Write-Host ""
                        Write-Host "Archivos de datos:" -ForegroundColor White
                        Write-Host "  - $fileIncluirExcluir" -ForegroundColor Gray
                        Write-Host