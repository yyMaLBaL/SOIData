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
                        New-Item -ItemType Directory -Force -Path "${env.EXECUTION_FOLDER}"
                    """
                }
            }
        }
        
        stage('Ejecutar Pruebas SIN CSV') {
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
                                    --reporter-html-export "\$reportFile"
                            """
                        }
                    }
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