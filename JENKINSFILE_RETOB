pipeline {
    agent any

    environment {
        AWS_REGION          = 'us-east-1'
        APP_NAME            = 'todolist'
        REPO_URL            = 'https://github.com/anitap26/todolist_CP3.git'
        // Para producción usamos un nombre de stack distinto
        STACK_NAME          = 'todolist-production'
        S3_BUCKET           = 'aws-sam-cli-managed-default-samclisourcebucket-fnxysdhusbno'
        TEMPLATE_FILE       = '.aws-sam/build/template.yaml'
        DYNAMODB_TABLE_NAME = 'TodosDynamoDbTable'
    }

    stages {

        stage('Get Code') {
            steps {
                script {
                    echo "Descargando el código de la rama master..."
                    // Si no existe el repositorio en el workspace, clónalo; de lo contrario, actualízalo.
                    if (!fileExists('.git')) {
                        echo "No se encontró repositorio git. Clonando..."
                        sh "git clone ${REPO_URL} ."
                        sh "git checkout master"
                    } else {
                        echo "Repositorio encontrado. Actualizando rama master..."
                        sh "git checkout master"
                        sh "git pull origin master"
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    echo "Desplegando la aplicación en producción..."
                    sh """
                        echo "Instalando dependencias..."
                        pip install aws-sam-cli

                        echo "Verificando SAM CLI..."
                        sam --version

                        echo "Construyendo la aplicación con SAM..."
                        sam build

                        echo "Validando la plantilla de SAM..."
                        sam validate --region ${AWS_REGION}

                        echo "Desplegando en Production..."
                        sam deploy --stack-name ${STACK_NAME} \\
                                   --s3-bucket ${S3_BUCKET} \\
                                   --s3-prefix todo-list-aws \\
                                   --template-file ${TEMPLATE_FILE} \\
                                   --region ${AWS_REGION} \\
                                   --no-confirm-changeset \\
                                   --capabilities CAPABILITY_IAM \\
                                   --parameter-overrides Stage=production DynamoDbTableName=${DYNAMODB_TABLE_NAME} || true
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
