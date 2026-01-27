pipeline {
    agent any
    
    parameters {
        choice(
            name: 'COLLECTION_FOLDER',
            choices: ['ALL', 'AT', 'SS', 'CS', 'CER', 'CERCS'],
            description: 'Selecciona la carpeta a ejecutar'
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
        
        stage('Configurar Ejecución') {
            steps {
                script {
                    def dateTimeOutput = powershell(returnStdout: true, script: '''
                        $date = Get-Date -Format "yyyy-MM-dd"
                        $time = Get-Date -Format "HH-mm-ss"
                        Write-Output "${date}_${time}"
                    ''').trim()
                    
                    env.EXECUTION_FOLDER = "Ejecuciones\\ACHDATA\\Fecha_${dateTimeOutput.replace('_', '_Hora_')}"
                    env.EXECUTION_DATE = dateTimeOutput.split('_')[0]
                    env.EXECUTION_TIME = dateTimeOutput.split('_')[1]
                    
                    powershell """
                        Write-Host "Creando carpeta: ${env.EXECUTION_FOLDER}"
                        New-Item -ItemType Directory -Force -Path "${env.EXECUTION_FOLDER}"
                    """
                }
            }
        }
        
        stage('Ejecutar Pruebas') {
            steps {
                script {
                    def folders = params.COLLECTION_FOLDER == 'ALL' ? 
                        ['AT', 'SS', 'CS', 'CER', 'CERCS'] : 
                        [params.COLLECTION_FOLDER]
                    
                    folders.each { folder ->
                        stage("Ejecutar ${folder}") {
                            powershell """
                                \$folderName = "${folder}"
                                \$reportFile = "${env.EXECUTION_FOLDER}\\report_\${folderName}.html"
                                
                                Write-Host "Ejecutando: \${folderName}"
                                
                                newman run "Collection/ACHDATA - YY.postman_collection.json" `
                                    -e "Environment/ACHData QA.postman_environment.json" `
                                    --folder "\${folderName}" `
                                    --insecure `
                                    --reporters cli,html `
                                    --reporter-html-export "\${reportFile}" `
                                    --iteration-data "File/Incluir_Excluir Personas.csv" `
                                    --iteration-data "File/50 Registros.csv" `
                                    --iteration-data "File/cargue_masivo_usuarios.csv"
                            """
                        }
                    }
                }
            }
        }
        
        stage('Mostrar Resultados') {
            steps {
                powershell """
                    Write-Host "=== EJECUCIÓN COMPLETADA ==="
                    Write-Host "Fecha: ${env.EXECUTION_DATE}"
                    Write-Host "Hora: ${env.EXECUTION_TIME}"
                    Write-Host "Carpeta: ${env.EXECUTION_FOLDER}"
                    
                    \$htmlFiles = Get-ChildItem "${env.EXECUTION_FOLDER}" -Filter "*.html"
                    Write-Host "Reportes generados: \$(\$htmlFiles.Count)"
                    
                    foreach (\$file in \$htmlFiles) {
                        Write-Host "  - \$(\$file.Name)"
                    }
                """
            }
        }
    }
    
    post {
        always {
            echo "Proceso finalizado"
            archiveArtifacts artifacts: "${env.EXECUTION_FOLDER}/**/*", allowEmptyArchive: true
        }
    }
}