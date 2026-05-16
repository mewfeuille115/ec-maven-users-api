pipeline {
  agent any

  environment {
    // --- Build / Docker ---
    IMAGE_NAME       = 'mi-aplicacion-java'
    IMAGE_TAG        = 'latest'
    DOCKERFILE_PATH  = 'Dockerfile'
    DOCKER_CREDS     = credentials('jksdockerregistryus2p01')   // crea DOCKER_CREDS_USR / DOCKER_CREDS_PSW
    ACR_REGISTRY     = 'containerregistrylcrp.azurecr.io'

    // --- Azure ACI ---
    APP_NAME         = 'myapp'
    ACI_NAME         = 'aci-myapp'          // minúsculas
    RESOURCE_GROUP   = 'devops'         // ajústalo
    LOCATION         = 'eastus'             // ajústalo
    CONTAINER_PORT   = '8080'

    // --- Service Principal (Secret Text) ---
    AZ_CLIENT_ID       = credentials('az-client-id')
    AZ_CLIENT_SECRET   = credentials('az-client-secret')
    AZ_TENANT_ID       = credentials('az-tenant-id')
    AZ_SUBSCRIPTION_ID = credentials('az-subscription-id')
  }

  options { timestamps() }

  stages {

    stage('Compilar con Maven') {
      agent { docker { image 'maven:3.8.4-openjdk-17-slim' } }
      steps {
        sh 'mvn -B -DskipTests clean package'
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, onlyIfSuccessful: true
      }
    }

    stage('Build Image') {
      steps {
        copyArtifacts(
          projectName: env.JOB_NAME,
          selector: [$class: 'SpecificBuildSelector', buildNumber: "${env.BUILD_NUMBER}"],
          filter: 'target/*.jar',
          fingerprintArtifacts: true,
          flatten: true,
          target: 'target'
        )
        sh 'docker build -t ${ACR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} -f ${DOCKERFILE_PATH} .'
      }
    }

    stage('Publish Image') {
      steps {
        sh '''
          set -eux
          docker login ${ACR_REGISTRY} -u ${DOCKER_CREDS_USR} -p ${DOCKER_CREDS_PSW}
          docker tag ${ACR_REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} ${ACR_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}
          docker push ${ACR_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}
          docker logout
        '''
      }
    }

    stage('Deploy to Azure Container Instances') {
      when {
        expression {
          def b = (env.BRANCH_NAME ?: env.GIT_BRANCH ?: '').trim()
          b == 'main' || b == 'origin/main' || b == 'refs/heads/main' || b.endsWith('/main')
        }
      }
      agent {
        docker { image 'mcr.microsoft.com/azure-cli'; args '-u 0:0' }  // para evitar permisos
      }
      environment {
        IMAGE_REF        = "${ACR_REGISTRY}/${IMAGE_NAME}:${BUILD_NUMBER}"
        AZURE_CONFIG_DIR = "${WORKSPACE}/.azure"   // evita escribir en '/.azure'
        HOME             = "${WORKSPACE}"
      }
      steps {
        sh '''
          set -eux
          mkdir -p "$AZURE_CONFIG_DIR"
    
          # Login con Service Principal (usa forma con "=" por si la contraseña empieza con "-")
          az login --service-principal \
            --username="${AZ_CLIENT_ID}" \
            --password="${AZ_CLIENT_SECRET}" \
            --tenant="${AZ_TENANT_ID}"
          az account set --subscription "${AZ_SUBSCRIPTION_ID}"
    
          # RG idempotente
          az group create -n "${RESOURCE_GROUP}" -l "${LOCATION}"
    
          # Borrar si ya existe (ACI no actualiza in-place)
          if az container show -g "${RESOURCE_GROUP}" -n "${ACI_NAME}" >/dev/null 2>&1; then
            az container delete -g "${RESOURCE_GROUP}" -n "${ACI_NAME}" -y
          fi
    
          # Etiqueta DNS pública (minúsculas)
          DNS_LABEL=$(echo "aci-${APP_NAME}-${BUILD_NUMBER}" | tr '[:upper:]' '[:lower:]')
    
          az container create \
            --resource-group "${RESOURCE_GROUP}" \
            --name "${ACI_NAME}" \
            --image "${IMAGE_REF}" \
            --registry-login-server "${ACR_REGISTRY}" \
            --registry-username "${DOCKER_CREDS_USR}" \
            --registry-password "${DOCKER_CREDS_PSW}" \
            --cpu 1 --memory 1.5 \
            --restart-policy Always \
            --ip-address Public \
            --ports ${CONTAINER_PORT} \
            --dns-name-label "${DNS_LABEL}" \
            --os-type Linux                 # <<< clave para evitar osType=null
    
          FQDN=$(az container show -g "${RESOURCE_GROUP}" -n "${ACI_NAME}" --query "ipAddress.fqdn" -o tsv)
          echo "ACI URL: http://${FQDN}:${CONTAINER_PORT}"
        '''
      }
    }



  }

  post {
    success { echo 'OK: build, push y deploy a Azure Container Instances.' }
    failure { echo 'ERROR en el pipeline.' }
  }
}