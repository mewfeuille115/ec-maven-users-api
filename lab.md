# Laboratorio: CI/CD con GitHub Actions → Azure Container Instances

> **Objetivo:** Construir un pipeline completo que compile una aplicación Java con Maven, publique la imagen en Azure Container Registry (ACR) y la despliegue automáticamente en Azure Container Instances (ACI) usando GitHub Actions.

---

## Índice

1. [Pre-requisitos](#1-pre-requisitos)
2. [Crear Azure Container Registry (ACR)](#2-crear-azure-container-registry-acr)
3. [Crear el Service Principal en Azure](#3-crear-el-service-principal-en-azure)
4. [Asignar permisos al Service Principal](#4-asignar-permisos-al-service-principal)
5. [Configurar Secrets en GitHub](#5-configurar-secrets-en-github)
6. [Estructura del repositorio](#6-estructura-del-repositorio)
7. [Agregar el workflow al repositorio](#7-agregar-el-workflow-al-repositorio)
8. [Ejecutar el pipeline](#8-ejecutar-el-pipeline)
9. [Verificar el despliegue](#9-verificar-el-despliegue)
10. [Limpieza de recursos](#10-limpieza-de-recursos)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. Pre-requisitos

Antes de comenzar, asegúrate de tener:

| Requisito | Versión / Detalle |
|-----------|-------------------|
| Cuenta Azure | Con suscripción activa |
| Azure CLI | `>= 2.50` instalado localmente |
| Cuenta GitHub | Repositorio con el código Java |
| Docker | Instalado localmente (opcional, para pruebas) |
| Java + Maven | Para compilar localmente (opcional) |

### Instalar Azure CLI (si no lo tienes)

```bash
# Linux / WSL
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# macOS
brew install azure-cli

# Verificar instalación
az version
```

### Login en Azure

```bash
az login
# Se abrirá el navegador para autenticarte

# Verifica tu suscripción activa
az account show
```

---

## 2. Crear Azure Container Registry (ACR)

El ACR es el registro privado donde se almacenarán las imágenes Docker de tu aplicación.

### 2.1 Crear el Resource Group

```bash
az group create \
  --name mod3lab2 \
  --location eastus
```

**Parámetros:**
- `--name`: nombre del resource group (usa el mismo que está en el workflow)
- `--location`: región de Azure (`eastus`, `westus2`, `brazilsouth`, etc.)

### 2.2 Crear el ACR

```bash
az acr create \
  --resource-group mod3lab2 \
  --name laboratorio \
  --sku Basic \
  --admin-enabled true
```

> ⚠️ **Importante:** El nombre del ACR debe ser **globalmente único** en Azure. Si `laboratorio` ya existe, usa un nombre diferente como `laboratorio<tuinicial><año>`.

**Parámetros clave:**
- `--sku Basic`: tier más económico, suficiente para labs
- `--admin-enabled true`: habilita usuario/contraseña para autenticación desde ACI

### 2.3 Obtener las credenciales del ACR

```bash
# Obtener el servidor de login
az acr show \
  --name laboratorio \
  --query loginServer \
  --output tsv

# Obtener usuario y contraseña
az acr credential show \
  --name laboratorio \
  --output table
```

Anota estos valores, los necesitarás en el paso 5:
- **Login server**: `laboratorio.azurecr.io`
- **Username**: generalmente el nombre del ACR
- **Password**: la primera contraseña listada

---

## 3. Crear el Service Principal en Azure

El Service Principal (SP) es una identidad de aplicación que GitHub Actions usará para autenticarse en Azure y realizar el deploy.

### 3.1 Obtener el ID de tu suscripción

```bash
az account show --query id --output tsv
```

Guarda el resultado: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

### 3.2 Crear el Service Principal

```bash
az ad sp create-for-rbac \
  --name "sp-github-actions-lab" \
  --role Contributor \
  --scopes /subscriptions/<TU_SUBSCRIPTION_ID>/resourceGroups/mod3lab2 \
  --output json
```

Reemplaza `<TU_SUBSCRIPTION_ID>` con el valor obtenido en el paso anterior.

**Salida esperada:**

```json
{
  "appId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "displayName": "sp-github-actions-lab",
  "password": "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  "tenant": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

> 🔐 **Guarda estos valores de forma segura.** El `password` solo se muestra una vez.

| Campo JSON | Secret de GitHub |
|------------|-----------------|
| `appId`    | `AZ_CLIENT_ID`  |
| `password` | `AZ_CLIENT_SECRET` |
| `tenant`   | `AZ_TENANT_ID`  |
| (subscription) | `AZ_SUBSCRIPTION_ID` |

---

## 4. Asignar permisos al Service Principal

El SP necesita permisos adicionales para interactuar con el ACR.

### 4.1 Obtener el ID del ACR

```bash
ACR_ID=$(az acr show \
  --name laboratorio \
  --resource-group mod3lab2 \
  --query id \
  --output tsv)

echo $ACR_ID
```

### 4.2 Asignar rol AcrPull al Service Principal

```bash
az role assignment create \
  --assignee <APP_ID_DEL_SP> \
  --role AcrPull \
  --scope $ACR_ID
```

Reemplaza `<APP_ID_DEL_SP>` con el `appId` obtenido en el paso 3.

> 💡 **¿Por qué AcrPull?** Aunque el workflow usa las credenciales admin del ACR para el push y el deploy, es buena práctica asignar también este rol al SP.

### 4.3 Verificar los roles asignados

```bash
az role assignment list \
  --assignee <APP_ID_DEL_SP> \
  --output table
```

---

## 5. Configurar Secrets en GitHub

Los secrets protegen tus credenciales; nunca se exponen en los logs del workflow.

### 5.1 Navegar a los Secrets del repositorio

1. Ve a tu repositorio en GitHub
2. Click en **Settings** (⚙️)
3. En el menú lateral: **Secrets and variables → Actions**
4. Click en **New repository secret**

### 5.2 Crear los siguientes secrets

Agrega cada uno de estos secrets **uno por uno**:

| Nombre del Secret | Valor |
|-------------------|-------|
| `AZ_CLIENT_ID` | `appId` del Service Principal |
| `AZ_CLIENT_SECRET` | `password` del Service Principal |
| `AZ_TENANT_ID` | `tenant` del Service Principal |
| `AZ_SUBSCRIPTION_ID` | ID de tu suscripción Azure |
| `DOCKER_CREDS_USR` | Usuario del ACR (nombre del registry) |
| `DOCKER_CREDS_PSW` | Contraseña del ACR |

### 5.3 Verificar los secrets creados

Deberías ver los 6 secrets listados (sin mostrar sus valores):

```
AZ_CLIENT_ID        ✓ Updated X minutes ago
AZ_CLIENT_SECRET    ✓ Updated X minutes ago
AZ_TENANT_ID        ✓ Updated X minutes ago
AZ_SUBSCRIPTION_ID  ✓ Updated X minutes ago
DOCKER_CREDS_USR    ✓ Updated X minutes ago
DOCKER_CREDS_PSW    ✓ Updated X minutes ago
```

> 💡 **Tip:** Si necesitas actualizar el nombre del ACR registry, lo puedes manejar como variable de entorno en el workflow directamente (está en la sección `env:` del archivo `.yml`).

---

## 6. Estructura del repositorio

El repositorio debe tener la siguiente estructura mínima:

```
tu-repositorio/
├── .github/
│   └── workflows/
│       └── ci-cd-azure-aci.yml     ← El workflow
├── src/
│   └── main/
│       └── java/
│           └── com/ejemplo/
│               └── App.java
├── Dockerfile                       ← Imagen de la app
├── pom.xml                          ← Configuración Maven
└── README.md
```

### 6.1 Ejemplo de Dockerfile para Java

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 6.2 Ejemplo mínimo de pom.xml

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.ejemplo</groupId>
  <artifactId>mi-aplicacion-java</artifactId>
  <version>1.0.0</version>
  <packaging>jar</packaging>

  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <configuration>
          <archive>
            <manifest>
              <mainClass>com.ejemplo.App</mainClass>
            </manifest>
          </archive>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

---

## 7. Agregar el workflow al repositorio

### 7.1 Crear la carpeta de workflows

```bash
mkdir -p .github/workflows
```

### 7.2 Copiar el archivo del workflow

Coloca el archivo `ci-cd-azure-aci.yml` en `.github/workflows/`.

### 7.3 Ajustar variables de entorno en el workflow

Abre el archivo y verifica/ajusta estos valores en la sección `env:`:

```yaml
env:
  IMAGE_NAME:      mi-aplicacion-java   # Nombre de tu imagen
  ACR_REGISTRY:    laboratorio.azurecr.io  # Tu login server del ACR
  ACI_NAME:        aci-myapp            # Nombre del contenedor en ACI
  RESOURCE_GROUP:  mod3lab2             # Tu resource group
  LOCATION:        eastus               # Región Azure
  CONTAINER_PORT:  "8080"              # Puerto de tu app
```

### 7.4 Hacer commit y push

```bash
git add .github/workflows/ci-cd-azure-aci.yml
git commit -m "ci: add GitHub Actions workflow for Azure ACI deploy"
git push origin main
```

---

## 8. Ejecutar el pipeline

### 8.1 Ejecución automática

Al hacer `push` a `main`, el workflow se disparará automáticamente. Puedes verlo en:

`GitHub → tu-repo → Actions → CI/CD → Azure Container Instances`

### 8.2 Ejecución manual

También puedes dispararlo desde la UI:

1. Ve a **Actions** en tu repositorio
2. Selecciona el workflow **CI/CD → Azure Container Instances**
3. Click en **Run workflow**
4. Selecciona la rama `main`
5. Click en **Run workflow** (botón verde)

### 8.3 Flujo de los jobs

```
┌─────────────────────┐
│  build-maven        │  ← Compila con Maven, guarda el JAR
└─────────┬───────────┘
          │ artifact: app-jar
          ▼
┌─────────────────────┐
│  build-image        │  ← Docker build, guarda la imagen como tar
└─────────┬───────────┘
          │ artifact: docker-image
          ▼
┌─────────────────────┐
│  publish-image      │  ← Push al ACR con tag :run_number
└─────────┬───────────┘
          │ (solo en rama main)
          ▼
┌─────────────────────┐
│  deploy-aci         │  ← Deploy en Azure Container Instances
└─────────────────────┘
```

---

## 9. Verificar el despliegue

### 9.1 Desde los logs de GitHub Actions

Al finalizar el job `deploy-aci`, en el paso **"Mostrar URL del contenedor"** verás:

```
✅ ACI URL: http://aci-myapp-42.eastus.azurecontainer.io:8080
```

### 9.2 Desde Azure CLI

```bash
# Ver estado del contenedor
az container show \
  --resource-group mod3lab2 \
  --name aci-myapp \
  --output table

# Ver los logs del contenedor en tiempo real
az container logs \
  --resource-group mod3lab2 \
  --name aci-myapp \
  --follow

# Obtener la URL pública
az container show \
  --resource-group mod3lab2 \
  --name aci-myapp \
  --query "ipAddress.fqdn" \
  --output tsv
```

### 9.3 Desde Azure Portal

1. Ve a [portal.azure.com](https://portal.azure.com)
2. Busca **Container instances**
3. Selecciona `aci-myapp`
4. En la columna **FQDN** verás la URL pública

---

## 10. Limpieza de recursos

Para evitar costos innecesarios después del laboratorio:

```bash
# Eliminar solo el contenedor
az container delete \
  --resource-group mod3lab2 \
  --name aci-myapp \
  --yes

# Eliminar el ACR (y todas sus imágenes)
az acr delete \
  --resource-group mod3lab2 \
  --name laboratorio \
  --yes

# Eliminar el Resource Group completo (elimina TODO)
az group delete \
  --name mod3lab2 \
  --yes \
  --no-wait

# Eliminar el Service Principal
az ad sp delete --id <APP_ID_DEL_SP>
```

> ⚠️ **Precaución:** `az group delete` elimina **todos** los recursos del resource group de forma irreversible.

---

## 11. Troubleshooting

### Error: `DOCKER_CREDS_PSW: unbound variable`

**Causa:** Las credenciales del ACR no están configuradas como secrets en GitHub.  
**Solución:** Verifica que los secrets `DOCKER_CREDS_USR` y `DOCKER_CREDS_PSW` estén creados en Settings → Secrets.

---

### Error: `unauthorized: authentication required` en el push al ACR

**Causa:** Contraseña incorrecta o el admin no está habilitado en el ACR.  
**Solución:**
```bash
# Re-habilitar admin y regenerar contraseña
az acr update --name laboratorio --admin-enabled true
az acr credential renew --name laboratorio --password-name password
az acr credential show --name laboratorio
```

---

### Error: `The subscription is not registered to use namespace 'Microsoft.ContainerInstance'`

**Causa:** El proveedor de ACI no está registrado en tu suscripción.  
**Solución:**
```bash
az provider register --namespace Microsoft.ContainerInstance
az provider show --namespace Microsoft.ContainerInstance --query registrationState
# Espera hasta ver "Registered"
```

---

### Error en `az login`: `AADSTS70011: The provided value for the input parameter 'scope' is not valid`

**Causa:** El `AZ_CLIENT_ID`, `AZ_CLIENT_SECRET` o `AZ_TENANT_ID` son incorrectos.  
**Solución:** Verifica que los secrets en GitHub coincidan exactamente con los valores del SP. Regenera el secret si es necesario:
```bash
az ad sp credential reset --id <APP_ID_DEL_SP>
```

---

### El job `deploy-aci` no se ejecuta en PRs

**Comportamiento esperado.** El deploy solo ocurre en pushes a `main` por la condición:
```yaml
if: github.ref == 'refs/heads/main'
```
Los Pull Requests ejecutan los jobs de build y publish, pero no el deploy.

---

### El contenedor arranca pero no responde en el puerto 8080

**Diagnóstico:**
```bash
# Ver logs del contenedor
az container logs --resource-group mod3lab2 --name aci-myapp

# Ver eventos del contenedor
az container show \
  --resource-group mod3lab2 \
  --name aci-myapp \
  --query "containers[0].instanceView.events" \
  --output table
```

**Causas comunes:**
- La app no escucha en `0.0.0.0` (debe escuchar en todas las interfaces, no solo `localhost`)
- El puerto en el `Dockerfile` (`EXPOSE`) no coincide con `CONTAINER_PORT`
- Error de configuración de Spring Boot: verificar `server.port=8080` en `application.properties`

---

## Referencias

- [GitHub Actions – Documentación oficial](https://docs.github.com/en/actions)
- [Azure Container Registry – Quickstart](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-get-started-azure-cli)
- [Azure Container Instances – Deploy with CLI](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-quickstart)
- [Azure Service Principals](https://learn.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal)
- [GitHub Encrypted Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
