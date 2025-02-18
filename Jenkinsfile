pipeline {
    agent any
    
    environment {
        AWS_REGION       = 'us-east-1'
        APP_NAME         = 'todolist'
        REPO_URL         = 'https://github.com/anitap26/todolist_CP3.git'
        // Stack para staging
        STACK_NAME       = 'todolist-staging'
        // URL base para descargar el samconfig.toml configurado para staging
        CONFIG_REPO_BASE = 'https://raw.githubusercontent.com/anitap26/todo-list-aws-config/staging'
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
                    
                    echo "Descargando archivo de configuración samconfig.toml para staging..."
                    sh "wget ${CONFIG_REPO_BASE}/samconfig.toml -O samconfig.toml"
                }
            }
        }

        stage('Static Test') {
            steps {
                script {
                    sh """
                        pip install flake8 bandit
                        export PATH=\$PATH:\$HOME/.local/bin
                        mkdir -p reports
                        flake8 src --output-file=reports/flake8_report.txt || true
                        bandit -r src -f json -o reports/bandit_report.json || true
                        touch reports/dummy.txt
                    """
                    sh "ls -la reports/"
                    sh "echo 'Contenido de flake8_report.txt:'; cat reports/flake8_report.txt || true"
                    sh "echo 'Contenido de bandit_report.json:'; cat reports/bandit_report.json || true"
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: '**/reports/*', fingerprint: true
                }
            }
        }

        stage('Generate Unique Table Name') {
            steps {
                script {
                    echo "Generando nombre único para la tabla DynamoDB..."
                    def uniqueId = sh(script: "echo \$(date +%s) | sha256sum | head -c 8", returnStdout: true).trim()
                    env.DYNAMODB_TABLE_NAME = "TodosDynamoDbTable${uniqueId}"
                    echo "Nombre de tabla generado: ${env.DYNAMODB_TABLE_NAME}"
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
                        sam deploy --stack-name ${STACK_NAME} \\
                                   --s3-bucket ${S3_BUCKET} \\
                                   --s3-prefix todo-list-aws \\
                                   --template-file ${TEMPLATE_FILE} \\
                                   --region ${AWS_REGION} \\
                                   --no-confirm-changeset \\
                                   --capabilities CAPABILITY_IAM \\
                                   --parameter-overrides Stage=staging DynamoDbTableName=${DYNAMODB_TABLE_NAME} || true
                    """
                }
            }
        }

        stage('Get API URL') {
            steps {
                script {
                    echo 'Obteniendo URL de la API desplegada...'
                    def apiUrl = sh(script: "aws cloudformation describe-stacks --stack-name ${STACK_NAME} --query \"Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue\" --output text --region ${AWS_REGION}", returnStdout: true).trim()
                    
                    if (!apiUrl || apiUrl == "None") {
                        error("No se pudo obtener la URL de la API desde CloudFormation. Verifica que el API Gateway se haya creado correctamente.")
                    }
                    echo "API URL obtenida: ${apiUrl}"
                    env.API_URL = apiUrl
                }
            }
        }

        stage('Pytest Rest Test') {
            steps {
                script {
                    if (!env.API_URL) {
                        error("No se pudo obtener la URL de la API. Fallando el pipeline.")
                    }
                    echo "Ejecutando pruebas REST con Pytest, utilizando BASE_URL=${env.API_URL}"
                    sh """
                        export PATH=\$PATH:/var/lib/jenkins/.local/bin
                        mkdir -p reports
                        # Ejecutar pytest apuntando al archivo ubicado en test/integration/todoApiTest.py
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

        stage('Promote') {
            steps {
                script {
                    echo "Actualizando la rama master..."
                    sh "git checkout master"
                    sh "git fetch origin master"
                    sh "git merge origin/master"
                    echo "Fusionando develop en master..."
                    sh "git merge develop"
                    echo "Empujando cambios a GitHub..."
                    withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                        sh '''#!/bin/bash
                        git push https://x-access-token:$GITHUB_TOKEN@github.com/anitap26/todolist_CP3.git master
                        '''
                    }
                }
            }
        }
    }
}
