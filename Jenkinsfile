pipeline {
    agent {
        docker { image 'devops-agent:latest' }
    }

    environment {
        // =========================
        // Ajusta estos valores
        // =========================
        APELLIDO = "achavez" // Cambiar por apellido
        ACR_NAME = "acrglobalcicd"
        ACR_LOGIN_SERVER = "${ACR_NAME}.azurecr.io"
        IMAGE_NAME = "my-nodejs-app-${APELLIDO}"
        RESOURCE_GROUP = "rg-cicd-terraform-app-baraujox"
        AKS_NAME = "aks-dev-eastus"

        // URLs por ambiente (ajusta a tu realidad o deja como demo)
        API_PROVIDER_URL_DEV = "https://dev.api.com"
        API_PROVIDER_URL_QA  = "https://qa.api.com"
        API_PROVIDER_URL_PRD = "https://prd.api.com"
    }

    stages {

        stage('[CI] Instalar dependencias de app (npm install)') {
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
                  if npm run | grep -qE "test:unit"; then
                    npm run test:unit
                  elif npm run | grep -qE "test"; then
                    npm test
                  else
                    echo ">>> No se encontró script de pruebas unitarias (test:unit / test). Se omite."
                  fi
                '''
            }
        }

        stage('[CI] Ejecutar pruebas de integración') {
            steps {
                sh '''
                  echo ">>> Ejecutando pruebas de integración..."
                  if npm run | grep -qE "test:integration"; then
                    npm run test:integration
                  else
                    echo ">>> No se encontró script test:integration. Se omite."
                  fi
                '''
            }
        }

        stage('[CI] Azure Login') {
            steps {
                withCredentials([
                    string(credentialsId: 'azure-clientId',       variable: 'AZ_CLIENT_ID'),
                    string(credentialsId: 'azure-clientSecret',   variable: 'AZ_CLIENT_SECRET'),
                    string(credentialsId: 'azure-tenantId',       variable: 'AZ_TENANT_ID'),
                    string(credentialsId: 'azure-subscriptionId', variable: 'AZ_SUBSCRIPTION_ID')
                ]) {
                    sh '''
                      echo ">>> Azure login (Service Principal)..."
                      az login --service-principal \
                        --username="$AZ_CLIENT_ID" \
                        --password="$AZ_CLIENT_SECRET" \
                        --tenant="$AZ_TENANT_ID"

                      az account set --subscription "$AZ_SUBSCRIPTION_ID"
                    '''
                }
            }
        }

        stage('[CI] AKS Credentials') {
            steps {
                sh '''
                  echo ">>> Obteniendo credenciales de AKS..."
                  az aks get-credentials \
                    --resource-group "$RESOURCE_GROUP" \
                    --name "$AKS_NAME" \
                    --overwrite-existing
                '''
            }
        }

        stage('[CI] Generar ID corto del commit') {
            steps {
                script {
                    env.IMAGE_TAG = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    echo ">>> IMAGE_TAG generado: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('[CI] Build and Push Docker Image') {
            steps {
                sh '''
                  echo ">>> Login al ACR..."
                  az acr login --name "$ACR_NAME"

                  echo ">>> Build de imagen..."
                  docker build -t "$ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG" .

                  echo ">>> Push al ACR..."
                  docker push "$ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG"
                '''
            }
        }

        // =========================
        // CD - DEV
        // =========================
        stage('[CD-DEV] Deploy a AKS') {
            steps {
                script {
                    env.ENV = "dev"
                    env.API_PROVIDER_URL = env.API_PROVIDER_URL_DEV
                }

                sh '''
                  echo ">>> Renderizando manifiesto para DEV..."
                  envsubst < k8s.yml > k8s-dev.yml
                  echo ">>> Aplicando en AKS (DEV)..."
                  kubectl apply -f k8s-dev.yml
                '''
            }
        }

        stage('[CD-DEV] Imprimir IP del servicio') {
            steps {
                sh '''
                  echo ">>> Obteniendo IP del LoadBalancer (DEV)..."
                  SERVICE_NAME="my-nodejs-service-${APELLIDO}-${ENV}"
                  LB_IP=""
                  MAX_RETRIES=30
                  RETRY_COUNT=0

                  while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                    LB_IP=$(kubectl get svc "$SERVICE_NAME" -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)
                    if [ -z "$LB_IP" ]; then
                      RETRY_COUNT=$((RETRY_COUNT+1))
                      echo "Intento $RETRY_COUNT/$MAX_RETRIES: IP aún no asignada, esperando 10s..."
                      sleep 10
                    fi
                  done

                  if [ -z "$LB_IP" ]; then
                    echo ">>> No se pudo obtener la IP del LoadBalancer en DEV."
                    exit 1
                  else
                    echo ">>> IP DEV: $LB_IP"
                  fi
                '''
            }
        }

        // =========================
        // Aprobación QA
        // =========================
        stage('Aprobación QA') {
            steps {
                input message: '¿Aprobar despliegue a QA?', ok: 'Aprobar QA'
            }
        }

        // =========================
        // CD - QA
        // =========================
        stage('[CD-QA] Deploy a AKS') {
            steps {
                script {
                    env.ENV = "qa"
                    env.API_PROVIDER_URL = env.API_PROVIDER_URL_QA
                }

                sh '''
                  echo ">>> Renderizando manifiesto para QA..."
                  envsubst < k8s.yml > k8s-qa.yml
                  echo ">>> Aplicando en AKS (QA)..."
                  kubectl apply -f k8s-qa.yml
                '''
            }
        }

        stage('[CD-QA] Imprimir IP del servicio') {
            steps {
                sh '''
                  echo ">>> Obteniendo IP del LoadBalancer (QA)..."
                  SERVICE_NAME="my-nodejs-service-${APELLIDO}-${ENV}"
                  LB_IP=""
                  MAX_RETRIES=30
                  RETRY_COUNT=0

                  while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                    LB_IP=$(kubectl get svc "$SERVICE_NAME" -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)
                    if [ -z "$LB_IP" ]; then
                      RETRY_COUNT=$((RETRY_COUNT+1))
                      echo "Intento $RETRY_COUNT/$MAX_RETRIES: IP aún no asignada, esperando 10s..."
                      sleep 10
                    fi
                  done

                  if [ -z "$LB_IP" ]; then
                    echo ">>> No se pudo obtener la IP del LoadBalancer en QA."
                    exit 1
                  else
                    echo ">>> IP QA: $LB_IP"
                  fi
                '''
            }
        }

        // =========================
        // Aprobación PRD
        // =========================
        stage('Aprobación PRD') {
            steps {
                input message: '¿Aprobar despliegue a PRD?', ok: 'Aprobar PRD'
            }
        }

        // =========================
        // CD - PRD
        // =========================
        stage('[CD-PRD] Deploy a AKS') {
            steps {
                script {
                    env.ENV = "prd"
                    env.API_PROVIDER_URL = env.API_PROVIDER_URL_PRD
                }

                sh '''
                  echo ">>> Renderizando manifiesto para PRD..."
                  envsubst < k8s.yml > k8s-prd.yml
                  echo ">>> Aplicando en AKS (PRD)..."
                  kubectl apply -f k8s-prd.yml
                '''
            }
        }

        stage('[CD-PRD] Imprimir IP del servicio') {
            steps {
                sh '''
                  echo ">>> Obteniendo IP del LoadBalancer (PRD)..."
                  SERVICE_NAME="my-nodejs-service-${APELLIDO}-${ENV}"
                  LB_IP=""
                  MAX_RETRIES=30
                  RETRY_COUNT=0

                  while [ -z "$LB_IP" ] && [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
                    LB_IP=$(kubectl get svc "$SERVICE_NAME" -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null || true)
                    if [ -z "$LB_IP" ]; then
                      RETRY_COUNT=$((RETRY_COUNT+1))
                      echo "Intento $RETRY_COUNT/$MAX_RETRIES: IP aún no asignada, esperando 10s..."
                      sleep 10
                    fi
                  done

                  if [ -z "$LB_IP" ]; then
                    echo ">>> No se pudo obtener la IP del LoadBalancer en PRD."
                    exit 1
                  else
                    echo ">>> IP PRD: $LB_IP"
                  fi
                '''
            }
        }
    }
}