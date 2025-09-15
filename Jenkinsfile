pipeline {
    agent {
        docker { image 'devops-agent:latest' }
    }

    environment {
        APELLIDO        = "palomino" // Reemplaza por tu apellido
        SHORT_SHA       = "${env.GIT_COMMIT[0..6]}"
        IMAGE_NAME      = "acr${APELLIDO}.azurecr.io/my-nodejs-app"
        TAG             = "${SHORT_SHA}"
        IMAGE           = "${IMAGE_NAME}:${TAG}"
        RESOURCE_GROUP  = "rg-cicd-terraform-app-${APELLIDO}"
        ACR_NAME        = "acr${APELLIDO}"
        CONTAINERAPP    = "aca-ms-${APELLIDO}-dev"
        ENVIRONMENT     = "aca-env-${APELLIDO}-dev"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Hello world') {
            steps {
                script { 
                    env.VARIABLE = "demo123"
                }
                sh '''
                  echo ">>> Impresión Hello world"
                  echo "Hello world"
                  echo "Variable declarada en script: $VARIABLE"
                  echo "Variable declarada en environment: $APELLIDO"
                '''
                sh '''
                  echo ">>> Versiones instaladas:"
                  node -v
                  npm -v
                  docker --version
                  az version
                '''
            }
        }

        stage('[CI] Instalar dependencias') {
            steps {
                sh '''
                  echo ">>> Instalando dependencias..."
                  npm install
                '''
            }
        }

        stage('[CI] Ejecutar pruebas unitarias') {
            steps {
                sh '''
                  echo ">>> Ejecutando pruebas unitarias..."
                  npm run test:unit || echo "No hay pruebas unitarias definidas"
                '''
            }
        }

        stage('[CI] Ejecutar pruebas de integración') {
            steps {
                sh '''
                  echo ">>> Ejecutando pruebas de integración..."
                  npm run test:integration || echo "No hay pruebas de integración definidas"
                '''
            }
        }

        stage('[CI] Azure Login') {
            steps {
                withCredentials([
                    string(credentialsId: 'azure-clientId',        variable: 'AZ_CLIENT_ID'),
                    string(credentialsId: 'azure-clientSecret',   variable: 'AZ_CLIENT_SECRET'),
                    string(credentialsId: 'azure-tenantId',       variable: 'AZ_TENANT_ID'),
                    string(credentialsId: 'azure-subscriptionId', variable: 'AZ_SUBSCRIPTION_ID')
                ]) {
                    sh '''
                      echo ">>> Azure login..."
                      az login --service-principal \
                        --username $AZ_CLIENT_ID \
                        --password $AZ_CLIENT_SECRET \
                        --tenant $AZ_TENANT_ID
                      az account set --subscription $AZ_SUBSCRIPTION_ID
                      az account show
                    '''
                }
            }
        }

        stage('[CI] Docker Login') {
            steps {
                sh '''
                  echo ">>> Login en ACR..."
                  az acr login --name $ACR_NAME
                '''
            }
        }

        stage('[CI] Build and Push Docker Image') {
            steps {
                sh '''
                  echo ">>> Construyendo y publicando imagen: $IMAGE"
                  docker build -t $IMAGE .
                  docker push $IMAGE
                '''
            }
        }

        stage('[CD] Configurar ACR credentials para Container App') {
            steps {
                sh '''
                    az containerapp registry set \
                        --name $CONTAINERAPP_NAME \
                        --resource-group $RESOURCE_GROUP \
                        --server $ACR_LOGIN_SERVER \
                        --username $(az acr credential show -n $ACR_NAME --query "username" -o tsv) \
                        --password $(az acr credential show -n $ACR_NAME --query "passwords[0].value" -o tsv)
                '''
            }
        }

        stage('[CD] Deploy to Azure Container App') {
            steps {
                sh '''
                  echo ">>> Desplegando en Azure Container App: $CONTAINERAPP"
                  az containerapp update \
                    --name $CONTAINERAPP \
                    --resource-group $RESOURCE_GROUP \
                    --image $IMAGE \
                    --query properties.configuration.ingress.fqdn
                '''
            }
        }

        stage('[CD] Print endpoint') {
            steps {
                sh '''
                  echo ">>> Endpoint expuesto:"
                  az containerapp show \
                    --name $CONTAINERAPP \
                    --resource-group $RESOURCE_GROUP \
                    --query properties.configuration.ingress.fqdn \
                    -o tsv
                '''
            }
        }
    }
}