pipeline {
    agent any

    environment {
        AWS_REGION       = 'us-east-1'
        APP_NAME         = 'todolist'
        REPO_URL         = 'https://github.com/anitap26/todolist_CP3.git'
        STACK_NAME       = 'todolist-staging'
        CONFIG_REPO_BASE = 'https://raw.githubusercontent.com/anitap26/todo-list-aws-config/production'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Get Code') {
            steps {
                script {
                    echo "Descargando el código de la rama develop..."
                    if (!fileExists('.git')) {
                        sh "git clone ${REPO_URL} ."
                        sh "git checkout develop"
                    } else {
                        sh "git checkout develop"
                        sh "git pull origin develop"
                    }
                    
                    echo "Descargando archivo de configuración samconfig.toml para production..."
                    sh "wget ${CONFIG_REPO_BASE}/samconfig.toml -O samconfig.toml"
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh """
                        echo "Instalando dependencias..."
                        pip install aws-sam-cli
                        echo "Verificando SAM CLI..."
                        sam --version
                        echo "Construyendo la aplicación con SAM..."
                        sam build
                        echo "Validando la plantilla de SAM..."
                        sam validate --region ${AWS_REGION}
                        echo "Desplegando en Staging..."
                        sam deploy --config-file samconfig.toml --config-env production \\
                                   --no-confirm-changeset \\
                                   --capabilities CAPABILITY_IAM || true
                    """
                }
            }
        }

        stage('Rest Test') {
            steps {
                script {
                    echo "Obteniendo la URL de la API de producción..."
                    // Extrae la URL Base de la API desde los outputs de CloudFormation
                    def apiUrl = sh(
                        script: "aws cloudformation describe-stacks --stack-name ${STACK_NAME} --query \"Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue\" --output text --region ${AWS_REGION}",
                        returnStdout: true
                    ).trim()

                    if (!apiUrl || apiUrl == "None") {
                        error("No se pudo obtener la URL de la API de producción.")
                    }
                    echo "API URL obtenida: ${apiUrl}"
                    env.API_URL = apiUrl

                    // Ejecuta las pruebas REST mediante Pytest
                    echo "Ejecutando pruebas REST con Pytest, utilizando BASE_URL=${env.API_URL}"
                    sh """
                        export PATH=\$PATH:/var/lib/jenkins/.local/bin
                        mkdir -p reports
                        BASE_URL=${env.API_URL} pytest -v test/integration/todoApiTest.py --junitxml=reports/pytest_report.xml || exit 1
                    """
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'reports/pytest_report.xml', fingerprint: true
                }
            }
        }
    }
}
