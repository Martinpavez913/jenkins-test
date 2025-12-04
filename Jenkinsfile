pipeline {
    agent any

    environment {
        PROJECT_NAME = "pipeline-test"
        SONARQUBE_URL = "http://sonarqube:9000"
        SONARQUBE_TOKEN = "sqa_7d1249e9c90ae210b8b93ecad90254b47476dbf4"
        TARGET_URL = "http://172.29.132.58:5000"
    }

    stages {
        stage('Install Python') {
            steps {
                sh '''
                    apt update
                    apt install -y python3 python3-venv python3-pip
                '''
            }
        }
        
        stage('Setup Environment') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Python Security Audit') {
            steps {
                sh '''
                    . venv/bin/activate
                    # --- MODIFICADO: Instalamos pip-audit Y bandit ---
                    pip install pip-audit bandit
                    
                    mkdir -p dependency-check-report
                    
                    echo "--- Ejecutando PIP AUDIT ---"
                    pip-audit -r requirements.txt -f markdown -o dependency-check-report/pip-audit.md || true
                    
                    echo "--- Ejecutando BANDIT ---"
                    # --- MODIFICADO: Ejecución de Bandit ---
                    # Guardamos en JSON para facilitar integración futura o lectura
                    bandit -r . -f json -o dependency-check-report/bandit-report.json || true
                '''
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQubeScanner') {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=$PROJECT_NAME \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=$SONARQUBE_URL \
                                -Dsonar.login=$SONARQUBE_TOKEN
                        """
                    }
                }
            }
        }

        stage('Dependency Check') {
            environment {
                NVD_API_KEY = credentials('nvdApiKey')
            }
            steps {
                dependencyCheck additionalArguments: "--scan . --format HTML --out dependency-check-report --enableExperimental --enableRetired --nvdApiKey ${NVD_API_KEY}", odcInstallation: 'DependencyCheck'
            }
        }

        stage('Publish Reports') {
            steps {
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'dependency-check-report',
                    reportFiles: 'dependency-check-report.html',
                    reportName: 'OWASP Dependency Check Report'
                ])
                
                // OPCIONAL: Si tienes el plugin "Warnings Next Generation", 
                // descomenta las siguientes lineas para ver gráficos de Bandit:
                // recordIssues(tools: [bandit(pattern: 'dependency-check-report/bandit-report.json')])
            }
        }
    }

    // --- MODIFICADO: Bloque Post para guardar el reporte de Bandit ---
    post {
        always {
            // Esto guarda el archivo json para que lo puedas descargar desde Jenkins
            archiveArtifacts artifacts: 'dependency-check-report/bandit-report.json', allowEmptyArchive: true
        }
    }
}
