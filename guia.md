# 🚀 Guía Completa: Taller Evaluativo 2 — Jenkins CI/CD con Nexus y Smee.io
### StudyTrack · Ingeniería de Software V · Universidad Icesi

> **Todo corre localmente en Ubuntu.** No se necesita servidor remoto ni cuenta cloud.

---

## 📋 Tabla de Contenidos

1. [Pre-requisitos del sistema](#1-pre-requisitos-del-sistema)
2. [Arquitectura general del ecosistema](#2-arquitectura-general-del-ecosistema)
3. [Fase 1 — Verificar código base](#fase-1--verificar-código-base-tests-y-build)
4. [Fase 2 — Dockerfiles multi-stage + pom.xml + nginx.conf](#fase-2--dockerfiles-multi-stage--pomxml--nginxconf)
5. [Fase 3 — Infraestructura: Nexus + Jenkins + Smee.io](#fase-3--infraestructura-nexus--jenkins--smeeio)
6. [Fase 4 — Jenkinsfile + deploy/docker-compose.yml](#fase-4--jenkinsfile--deploydocker-composeyml)
7. [Prompts para IA (ChatGPT / Claude / Gemini)](#-prompts-para-ia-chatgpt--claude--gemini)
8. [Troubleshooting frecuente](#-troubleshooting-frecuente)
9. [Checklist final de entrega](#-checklist-final-de-entrega)

---

## 1. Pre-requisitos del sistema

### 1.1 Instalar Docker y Docker Compose en Ubuntu

```bash
# Actualizar repositorios
sudo apt-get update

# Instalar dependencias
sudo apt-get install -y ca-certificates curl gnupg lsb-release

# Añadir clave GPG de Docker
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Añadir repositorio Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker Engine + Compose plugin
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Agregar tu usuario al grupo docker (evita usar sudo en cada comando)
sudo usermod -aG docker $USER

# Reiniciar sesión o ejecutar para activar el grupo
newgrp docker

# Verificar instalación
docker --version
docker compose version
```

### 1.2 Instalar Java 17, Maven y Node.js (para pruebas locales en Fase 1)

```bash
# Java 17
sudo apt install -y openjdk-17-jdk
java -version

# Maven
sudo apt install -y maven
mvn -version

# Node.js 18 via nvm
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 18
nvm use 18
node --version
npm --version
```

### 1.3 Instalar Git

```bash
sudo apt install -y git
git --version
```

### 1.4 Puertos que usará el ecosistema (sugeridos, ajustables en `.env`)

| Servicio                | Puerto sugerido | Variable                     |
|-------------------------|-----------------|------------------------------|
| Jenkins UI              | `8080`          | `JENKINS_UI_PORT`            |
| Jenkins Agent TCP       | `50000`         | `JENKINS_AGENT_PORT`         |
| Nexus UI                | `8081`          | `NEXUS_UI_PORT`              |
| Nexus Docker Registry   | `5000`          | `NEXUS_DOCKER_REGISTRY_PORT` |
| Backend (app)           | `8082`          | —                            |
| Frontend (app)          | `3000`          | —                            |

> **Nota:** Si algún puerto ya está en uso, cámbialo en el `.env` correspondiente.
> Lo importante es la consistencia entre `docker-compose.yml` y `Jenkinsfile`.

---

## 2. Arquitectura general del ecosistema

```
┌─────────────────────────────────────────────────────────────┐
│                    Tu máquina Ubuntu                        │
│                                                             │
│  ┌──────────────┐   webhook   ┌─────────────────────┐      │
│  │   GitHub     │────────────▶│   smee.io (relay)   │      │
│  │ (repositorio)│            │   smee-client:local  │      │
│  └──────────────┘            └──────────┬──────────┘      │
│                                         │ HTTP interno     │
│  ┌──────────────────────────────────────▼──────────────┐   │
│  │           cicd_network (Docker bridge)               │   │
│  │                                                      │   │
│  │  ┌─────────────┐     ┌──────────────────────────┐   │   │
│  │  │   Jenkins   │────▶│        Nexus OSS          │   │   │
│  │  │  :8080      │     │  UI:8081 | Docker:5000    │   │   │
│  │  └─────┬───────┘     └──────────────────────────┘   │   │
│  │        │ docker.sock (socket del host)               │   │
│  │        ▼                                             │   │
│  │  ┌─────────────┐   ┌──────────────────────────┐    │   │
│  │  │  Backend    │   │       Frontend            │    │   │
│  │  │  :8082      │◀──│       :3000               │    │   │
│  │  └─────────────┘   └──────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Flujo CI/CD:**
1. Haces `git push` a GitHub
2. GitHub llama al canal de **smee.io**
3. `smee-client` (contenedor) retransmite el evento HTTP a Jenkins en la red interna
4. Jenkins ejecuta el **Jenkinsfile** con 4 etapas:
   - `Checkout & Test` → pruebas unitarias del backend
   - `Package & Tag Inmutable` → build de imágenes Docker con tag SHA del commit
   - `Publish to Nexus` → push de imágenes al Docker Registry de Nexus + JAR Maven
   - `Deploy & Smoke Test` → levanta la app con `docker compose` y valida que responde

---

## Fase 1 — Verificar código base (Tests y Build)

> **Objetivo:** Confirmar que el código base funciona antes de contenerizarlo.

```bash
# 1. Ir al directorio del proyecto
cd codigo_base

# 2. Ejecutar pruebas del backend
cd backend
mvn test
# Debe mostrar: BUILD SUCCESS con 2+ tests pasando

# 3. Build del backend (genera el JAR)
mvn package -DskipTests
ls target/studytrack-api.jar   # verificar que existe

# 4. Verificar el frontend
cd ../frontend
npm install
npm run build
ls dist/   # verificar que existe la carpeta dist
```

> La Fase 1 **no requiere modificar ningún archivo**. Solo verificar que el código base funciona.

---

## Fase 2 — Dockerfiles multi-stage + pom.xml + nginx.conf

> **Objetivo:** Contenerizar backend y frontend con imágenes inmutables multi-stage.

### 2.1 Backend Dockerfile (`codigo_base/backend/Dockerfile`)

Principios a cumplir:
- **Stage 1 (builder):** imagen con JDK + Maven para compilar
- **Stage 2 (runtime):** imagen solo con JRE, sin SDK ni compilador
- Usuario **no-root** en runtime
- Cache de capas: copiar `pom.xml` y descargar dependencias **antes** que el código fuente

```dockerfile
# ---------- Stage 1: Builder ----------
FROM eclipse-temurin:17-jdk-alpine AS builder

WORKDIR /app

# Cache de dependencias Maven (capa separada del código fuente)
COPY pom.xml .
RUN mvn dependency:go-offline -B --no-transfer-progress 2>/dev/null || true

# Copiar fuentes y compilar
COPY src ./src
RUN mvn package -DskipTests -B --no-transfer-progress

# ---------- Stage 2: Runtime ----------
FROM eclipse-temurin:17-jre-alpine

# Usuario no-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
WORKDIR /app

COPY --from=builder /app/target/studytrack-api.jar app.jar
RUN chown appuser:appgroup app.jar

USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

> **Por qué JRE y no JDK en runtime?**
> El JRE solo contiene lo necesario para *ejecutar* Java. El JDK incluye el compilador y herramientas de desarrollo. La imagen final es ~60% más pequeña.

### 2.2 Frontend Dockerfile (`codigo_base/frontend/Dockerfile`)

```dockerfile
# ---------- Stage 1: Builder ----------
FROM node:18-alpine AS builder

WORKDIR /app

# Cache de dependencias npm (capa separada del código)
COPY package*.json ./
RUN npm ci

# Copiar fuentes y construir
COPY . .
RUN npm run build

# ---------- Stage 2: Runtime ----------
FROM nginx:1.25-alpine

# Copiar el build de Vite al directorio de nginx
COPY --from=builder /app/dist /usr/share/nginx/html

# Copiar configuración personalizada
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 3000
CMD ["nginx", "-g", "daemon off;"]
```

### 2.3 Frontend nginx.conf (`codigo_base/frontend/nginx.conf`)

El "fallback SPA" redirige todas las rutas a `index.html` para que React Router funcione:

```nginx
server {
    listen 3000;
    server_name localhost;

    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Cache agresiva para assets con hash de contenido (Vite)
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

### 2.4 Backend pom.xml — bloque `distributionManagement`

Abre `codigo_base/backend/pom.xml` y realiza estos dos cambios:

**Cambio 1:** Versión fija (no SNAPSHOT):
```xml
<!-- Cambiar: -->
<version>0.0.1-SNAPSHOT</version>
<!-- Por: -->
<version>1.0.0</version>
```

**Cambio 2:** Agregar antes del cierre `</project>`:
```xml
<distributionManagement>
    <repository>
        <id>nexus-releases</id>
        <url>http://nexus:8081/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>nexus-snapshots</id>
        <url>http://nexus:8081/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

> **¿Por qué usar `nexus` como hostname y no `localhost`?**
> El Jenkinsfile corre dentro del contenedor de Jenkins. Desde ahí, `localhost` es el propio contenedor Jenkins. El nombre `nexus` resuelve al contenedor de Nexus gracias a la red `cicd_network` de Docker.

> **¿Por qué versión fija y no SNAPSHOT?**
> Nexus `maven-releases` rechaza artefactos con versión `-SNAPSHOT` por política. Usamos versión fija para garantizar **empaquetamiento inmutable**.

### 2.5 Verificar imágenes localmente

```bash
cd codigo_base

# Build y test del backend
docker build -t studytrack-api:local ./backend
docker run --rm -d -p 8082:8080 --name test-backend studytrack-api:local
sleep 10
curl http://localhost:8082/tasks   # debe responder []  o lista JSON
docker stop test-backend

# Build y test del frontend
docker build -t studytrack-frontend:local ./frontend
docker run --rm -d -p 3000:3000 --name test-frontend studytrack-frontend:local
sleep 5
curl -I http://localhost:3000      # debe responder HTTP/1.1 200 OK
docker stop test-frontend
```

---

## Fase 3 — Infraestructura: Nexus + Jenkins + Smee.io

> **Objetivo:** Levantar el ecosistema CI/CD completo en contenedores Docker.

### 3.1 Crear la red compartida

```bash
docker network create cicd_network
```

---

### 3.2 Nexus — `infra/nexus_config/`

**Paso 1:** Crear el archivo `.env`:
```bash
cd infra/nexus_config
cp .env.example .env
```

Editar `.env`:
```env
NEXUS_UI_PORT=8081
NEXUS_DOCKER_REGISTRY_PORT=5000
```

**Paso 2:** Completar `docker-compose.yml`:

```yaml
version: "3.9"

services:
  nexus:
    image: sonatype/nexus3:3.68.1
    container_name: nexus
    networks:
      - cicd_network
    ports:
      - "${NEXUS_UI_PORT:-8081}:8081"
      - "${NEXUS_DOCKER_REGISTRY_PORT:-5000}:5000"
    volumes:
      - nexus_data:/nexus-data
    environment:
      - INSTALL4J_ADD_VM_PARAMS=-Xms512m -Xmx1g -XX:MaxDirectMemorySize=2g
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8081/service/rest/v1/status"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 120s

networks:
  cicd_network:
    external: true

volumes:
  nexus_data:
```

**Paso 3:** Levantar Nexus:

```bash
docker compose up -d
# Esperar hasta 5 minutos (Nexus es lento al arrancar)
docker compose logs -f nexus   # buscar "Started Sonatype Nexus OSS"
```

**Paso 4:** Configurar Nexus en `http://localhost:8081`:

```bash
# Obtener la contraseña inicial de admin
docker exec nexus cat /nexus-data/admin.password
```

En la UI de Nexus:
1. Login: `admin` + contraseña obtenida → cambiar contraseña
2. **Deshabilitar Anonymous Access** en el wizard inicial
3. Crear repositorio Docker hosted:
   - `Repository > Repositories > Create repository > docker (hosted)`
   - Name: `docker-hosted`
   - HTTP port: `5000`
   - Allow anonymous pull: **OFF**
4. Crear repositorio Maven:
   - `Create repository > maven2 (hosted)`
   - Name: `maven-releases`
   - Version policy: **Release**
   - Deployment policy: **Allow redeploy**
5. Crear usuario para Jenkins:
   - `Security > Users > Create local user`
   - Username: `jenkins-user`, Password: (anotar)
   - Roles: `nx-admin`

**Paso 5:** Configurar Docker para aceptar el registry inseguro:

```bash
# Editar en el HOST de Ubuntu
sudo nano /etc/docker/daemon.json
```
```json
{
  "insecure-registries": ["localhost:5000"]
}
```
```bash
sudo systemctl restart docker
# Verificar
docker info | grep -A3 "Insecure Registries"
```

---

### 3.3 Jenkins — `infra/jenkins_config/`

**Paso 1:** Completar `Dockerfile` (instalar Docker CLI):

```dockerfile
FROM jenkins/jenkins:lts-jdk17

USER root

# Instalar Docker CLI para que Jenkins construya y publique imágenes
RUN apt-get update && \
    apt-get install -y ca-certificates curl gnupg lsb-release && \
    mkdir -p /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg | \
      gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
      https://download.docker.com/linux/debian \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      tee /etc/apt/sources.list.d/docker.list > /dev/null && \
    apt-get update && \
    apt-get install -y docker-ce-cli && \
    rm -rf /var/lib/apt/lists/*

# Agregar jenkins al grupo docker del host (GID 999 es el habitual en Ubuntu)
RUN groupadd -f -g 999 docker && usermod -aG docker jenkins

COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

USER jenkins
```

**Paso 2:** Crear el archivo Smee Dockerfile en `infra/jenkins_config/smee/Dockerfile`:

```dockerfile
FROM node:18-alpine
RUN npm install -g smee-client
ENTRYPOINT ["smee", "--url", "$SMEE_URL", "--target", "$TARGET_URL"]
```

O usando la forma de shell para expandir variables:

```dockerfile
FROM node:18-alpine
RUN npm install -g smee-client
CMD smee --url "$SMEE_URL" --target "$TARGET_URL"
```

**Paso 3:** Crear el archivo `.env`:

```bash
cp .env.example .env
```

Editar `.env`:
```env
JENKINS_UI_PORT=8080
JENKINS_AGENT_PORT=50000
SMEE_CHANNEL_URL=https://smee.io/TU_CANAL_AQUI
```

**Paso 4:** Completar `docker-compose.yml`:

```yaml
version: "3.9"

services:
  jenkins:
    build: .
    container_name: jenkins
    networks:
      - cicd_network
    ports:
      - "${JENKINS_UI_PORT:-8080}:8080"
      - "${JENKINS_AGENT_PORT:-50000}:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock

  smee:
    build: ./smee
    container_name: smee-client
    environment:
      - SMEE_URL=${SMEE_CHANNEL_URL}
      - TARGET_URL=http://jenkins:8080/github-webhook/
    networks:
      - cicd_network
    depends_on:
      - jenkins

networks:
  cicd_network:
    name: cicd_network

volumes:
  jenkins_home:
```

> **¿Por qué montar `/var/run/docker.sock`?**
> El socket de Docker es el canal de comunicación con el daemon de Docker del host. Al montarlo en Jenkins, el contenedor puede ejecutar comandos `docker build` y `docker push` usando el daemon del host. Esto evita Docker-in-Docker (DinD), que es más complejo y menos estable.
>
> **Riesgo de seguridad:** Quien controla el socket controla el daemon completo del host. En producción se usarían otras alternativas como Kaniko o buildah. En un entorno local de desarrollo, es aceptable.

**Paso 5:** Crear canal en Smee.io

1. Ve a [https://smee.io](https://smee.io)
2. Haz clic en **"Start a new channel"**
3. Copia la URL generada (ej: `https://smee.io/aBcDeFgHiJ123`)
4. Pégala en `SMEE_CHANNEL_URL` del `.env`

**Paso 6:** Configurar webhook en GitHub

1. En tu repositorio: `Settings > Webhooks > Add webhook`
2. **Payload URL:** la URL de smee.io
3. **Content type:** `application/json`
4. **Events:** Solo `push`
5. Guardar

**Paso 7:** Levantar Jenkins:

```bash
cd infra/jenkins_config
docker compose up -d --build
docker compose logs -f   # esperar hasta ver la contraseña inicial de Jenkins
```

**Paso 8:** Configurar Jenkins en `http://localhost:8080`:

```bash
# Obtener contraseña inicial
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

En la UI de Jenkins:
1. Ingresar contraseña inicial → instalar plugins sugeridos → crear admin
2. **Agregar credenciales de Nexus:**
   - `Manage Jenkins > Credentials > System > Global > Add Credentials`
   - Kind: **Username with password**
   - ID: `nexus-credentials` ← **exactamente este texto**
   - Username/Password: usuario `jenkins-user` creado en Nexus
3. **Crear el Pipeline:**
   - `New Item > Pipeline`
   - Nombre: `studytrack-pipeline`
   - Definition: **Pipeline script from SCM**
   - SCM: **Git**, URL del repositorio
   - Branch: `*/main`
   - Script Path: `codigo_base/Jenkinsfile`
   - ✅ **GitHub hook trigger for GITScm polling**

---

## Fase 4 — Jenkinsfile + deploy/docker-compose.yml

> **Objetivo:** Pipeline declarativo completo con 4 etapas.

### 4.1 `deploy/docker-compose.yml`

```yaml
version: "3.9"

services:
  backend:
    image: localhost:5000/studytrack-api:${IMAGE_TAG}
    container_name: studytrack-backend
    ports:
      - "8082:8080"
    networks:
      - product-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/tasks"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s

  frontend:
    image: localhost:5000/studytrack-frontend:${IMAGE_TAG}
    container_name: studytrack-frontend
    ports:
      - "3000:3000"
    networks:
      - product-network
    depends_on:
      backend:
        condition: service_healthy

networks:
  product-network:
    driver: bridge
```

> **¿Por qué `service_healthy` y no `service_started`?**
> `service_started` solo espera a que el proceso arranque. `service_healthy` espera a que el healthcheck pase, garantizando que el backend ya acepta peticiones antes de que el frontend intente conectarse.

### 4.2 `codigo_base/Jenkinsfile` completo

```groovy
// =============================================================================
// Taller Evaluativo 2 - Fase 4: Pipeline Declarativo en Jenkins
// Prohibido el tag :latest — tag inmutable = primeros 7 chars del SHA del commit
// Las credenciales se inyectan por withCredentials, NUNCA en texto plano.
// =============================================================================

pipeline {
    agent any

    environment {
        IMAGE_TAG            = ""
        NEXUS_REGISTRY       = "localhost:5000"
        NEXUS_MAVEN_REPO     = "http://nexus:8081/repository/maven-releases/"
        NEXUS_CREDENTIALS_ID = "nexus-credentials"
    }

    stages {

        stage('Checkout & Test') {
            steps {
                checkout scm

                script {
                    // Tag inmutable = primeros 7 caracteres del SHA del commit
                    IMAGE_TAG = sh(
                        script: "git rev-parse --short=7 HEAD",
                        returnStdout: true
                    ).trim()
                    env.IMAGE_TAG = IMAGE_TAG
                    echo "Tag de imagen inmutable: ${IMAGE_TAG}"
                }

                dir('codigo_base/backend') {
                    sh 'mvn test -B --no-transfer-progress'
                }
            }
        }

        stage('Package & Tag Inmutable') {
            steps {
                dir('codigo_base/backend') {
                    sh """
                        docker build \
                          -t ${NEXUS_REGISTRY}/studytrack-api:${IMAGE_TAG} \
                          .
                    """
                }
                dir('codigo_base/frontend') {
                    sh """
                        docker build \
                          -t ${NEXUS_REGISTRY}/studytrack-frontend:${IMAGE_TAG} \
                          .
                    """
                }
            }
        }

        stage('Publish to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${NEXUS_CREDENTIALS_ID}",
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    // Push imágenes Docker
                    sh """
                        echo "${NEXUS_PASS}" | \
                          docker login ${NEXUS_REGISTRY} \
                          -u ${NEXUS_USER} --password-stdin

                        docker push ${NEXUS_REGISTRY}/studytrack-api:${IMAGE_TAG}
                        docker push ${NEXUS_REGISTRY}/studytrack-frontend:${IMAGE_TAG}

                        docker logout ${NEXUS_REGISTRY}
                    """

                    // Deploy del JAR Maven a Nexus
                    dir('codigo_base/backend') {
                        sh """
                            mvn deploy \
                              -DskipTests \
                              -B --no-transfer-progress \
                              -DaltDeploymentRepository=nexus-releases::default::${NEXUS_MAVEN_REPO} \
                              -Dusername=${NEXUS_USER} \
                              -Dpassword=${NEXUS_PASS}
                        """
                    }
                }
            }
        }

        stage('Deploy & Smoke Test') {
            steps {
                dir('codigo_base/deploy') {
                    sh """
                        IMAGE_TAG=${IMAGE_TAG} docker compose up -d --pull always
                        sleep 45
                        curl -f http://localhost:8082/tasks || \
                          (docker compose logs && exit 1)
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline finalizado en verde. Artefactos publicados con tag: ${env.IMAGE_TAG}"
        }
        failure {
            echo "El pipeline falló. Revise los logs de la etapa correspondiente antes de reintentar."
        }
    }
}
```

---

## 🤖 Prompts para IA (ChatGPT / Claude / Gemini)

Copia y pega cada prompt directamente. Son autocontenidos y tienen todo el contexto necesario.

---

### PROMPT 1 — Dockerfiles Multi-Stage (Fase 2)

```
Eres un experto en DevOps y Docker. Estoy haciendo un taller universitario de CI/CD.

Tengo una aplicación llamada StudyTrack con:
- Backend: Spring Boot 3 con Java 17, construido con Maven (mvn package genera target/studytrack-api.jar)
- Frontend: React 18 + Vite + TypeScript (npm run build genera la carpeta dist/)

Necesito que me escribas DOS Dockerfiles multi-stage para Ubuntu/Alpine Linux:

**Dockerfile del Backend** (archivo: codigo_base/backend/Dockerfile):
Requisitos:
1. Stage 1 (builder): usar eclipse-temurin:17-jdk-alpine, copiar pom.xml primero, luego
   mvn dependency:go-offline para cache de capas, luego copiar src/ y ejecutar mvn package -DskipTests
2. Stage 2 (runtime): usar eclipse-temurin:17-jre-alpine (sin JDK), crear usuario no-root
   llamado appuser, copiar el JAR del builder, exponer en el puerto 8080
3. La imagen final NO debe contener el JDK ni Maven

**Dockerfile del Frontend** (archivo: codigo_base/frontend/Dockerfile):
Requisitos:
1. Stage 1 (builder): usar node:18-alpine, copiar package*.json primero, npm ci para cache,
   luego copiar todo y npm run build
2. Stage 2 (runtime): usar nginx:1.25-alpine, copiar dist/ a /usr/share/nginx/html,
   copiar nginx.conf a /etc/nginx/conf.d/default.conf, exponer puerto 3000

**nginx.conf** (archivo: codigo_base/frontend/nginx.conf):
- Escuchar en puerto 3000
- Configurar fallback SPA con try_files $uri $uri/ /index.html
- Cache agresiva para /assets/ con expires 1y

Explica brevemente cada instrucción y por qué se hace en ese orden.
Dame también el comando docker build y docker run para probar cada imagen localmente.
```

---

### PROMPT 2 — pom.xml distributionManagement (Fase 2)

```
Soy estudiante de ingeniería de software. Tengo un proyecto Spring Boot 3 con Java 17
cuyo pom.xml actual tiene:
- groupId: com.icesi
- artifactId: studytrack-api
- version: 0.0.1-SNAPSHOT

Necesito:
1. Agregar el bloque <distributionManagement> para publicar el JAR en un servidor
   Nexus OSS 3 que corre localmente.
   - El backend de Jenkins accede a Nexus por el hostname "nexus" dentro de la red Docker cicd_network
   - URL del repositorio de releases: http://nexus:8081/repository/maven-releases/
   - URL del repositorio de snapshots: http://nexus:8081/repository/maven-snapshots/
   - El ID del repositorio de releases debe ser exactamente "nexus-releases"
2. Cambiar la versión de 0.0.1-SNAPSHOT a 1.0.0 (para publicación en maven-releases,
   ya que Nexus rechaza versiones SNAPSHOT en repositorios de release)
3. Explicar qué es un repositorio de artefactos y por qué usamos versión fija

Dame el bloque XML completo a agregar y también el comando mvn deploy con las
variables de autenticación para publicar en Nexus (las credenciales se inyectan
por variables de entorno en CI).
```

---

### PROMPT 3 — docker-compose.yml de Nexus (Fase 3)

```
Soy estudiante de DevOps. Necesito completar el docker-compose.yml de Sonatype Nexus OSS 3
para un entorno de CI/CD local en Ubuntu.

Contexto:
- Nexus corre en un contenedor Docker
- Debe conectarse a una red externa de Docker llamada "cicd_network" (ya existe)
- Debe exponer: puerto 8081 para la UI y puerto 5000 para el Docker Registry hosted
- Los puertos se leen de variables de entorno ${NEXUS_UI_PORT} y ${NEXUS_DOCKER_REGISTRY_PORT}
- Los datos deben persistir en un volumen Docker "nexus_data" mapeado a /nexus-data
- Necesita un healthcheck que verifique que Nexus responde en su endpoint de status
- Necesita memoria asignada via variable de entorno INSTALL4J_ADD_VM_PARAMS

Tengo este esqueleto incompleto:

version: "3.9"
services:
  nexus:
    image: sonatype/nexus3:3.68.1
    container_name: nexus
    networks:
      - cicd_network
    ports:
    volumes:
    environment:
    healthcheck:
      test:
      interval:
      timeout:
      retries:
      start_period:
networks:
  cicd_network:
    external: true
volumes:
  nexus_data:

Complétalo con todos los valores correctos. Explica por qué start_period debe ser largo
(Nexus tarda mucho en arrancar). Dame también los pasos para crear el repositorio
Docker hosted y el maven-releases en la UI de Nexus después de que levante.
```

---

### PROMPT 4 — docker-compose.yml de Jenkins + smee (Fase 3)

```
Soy estudiante de DevOps. Necesito completar el docker-compose.yml de Jenkins + smee-client
para un entorno de CI/CD local en Ubuntu.

Contexto:
- Jenkins corre desde un Dockerfile local (mismo directorio) que ya tiene Docker CLI instalado
- Jenkins debe construir imágenes Docker usando el socket del host (/var/run/docker.sock)
- Los datos de Jenkins persisten en un volumen "jenkins_home"
- Jenkins está en la red "cicd_network"
- Puertos: ${JENKINS_UI_PORT} para UI (8080 interno) y ${JENKINS_AGENT_PORT} (50000 interno)
- smee-client recibe webhooks de GitHub desde smee.io y los reenvía a Jenkins
- smee necesita variables: SMEE_URL (URL del canal) y TARGET_URL (http://jenkins:8080/github-webhook/)
- smee depende de que Jenkins esté corriendo

Tengo este esqueleto incompleto:

version: "3.9"
services:
  jenkins:
    build: .
    container_name:
    networks:
    ports:
    volumes:
  smee:
    build: ./smee
    container_name: smee-client
    environment:
    networks:
    depends_on:
networks:
  cicd_network:
    name: cicd_network
volumes:
  jenkins_home:

Complétalo. Explica por qué montar /var/run/docker.sock es necesario y qué
riesgo de seguridad implica. Explica también cómo smee-client funciona como
relay de webhooks entre GitHub e internet y Jenkins en local.
```

---

### PROMPT 5 — Jenkins Dockerfile (instalar Docker CLI) (Fase 3)

```
Tengo este Dockerfile de Jenkins que necesito completar:

FROM jenkins/jenkins:lts-jdk17

USER root

# TODO: instalar Docker CLI

COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

USER jenkins

El bloque TODO debe instalar el Docker CLI en la imagen de Jenkins (basada en Debian)
para que Jenkins pueda ejecutar comandos docker build, docker push, docker login
durante el pipeline. El daemon de Docker NO se instala dentro del contenedor — Jenkins
usa el socket del host montado como volumen.

Requisitos:
1. Instalar solo docker-ce-cli (no el daemon)
2. Agregar al usuario "jenkins" al grupo "docker" con GID 999 (GID habitual en Ubuntu)
3. Limpiar la caché de apt al final para reducir el tamaño de la imagen
4. Explicar por qué solo el CLI y no el daemon completo

Dame el Dockerfile completo y correcto.
```

---

### PROMPT 6 — Smee Dockerfile (relay de webhooks) (Fase 3)

```
Necesito crear el Dockerfile para smee-client, que es un contenedor que hace de relay
entre smee.io (servicio en internet) y Jenkins (en red Docker local).

El contenedor debe:
1. Usar node:18-alpine como base
2. Instalar el paquete npm "smee-client" globalmente
3. Al ejecutarse, correr el comando: smee --url $SMEE_URL --target $TARGET_URL
   donde SMEE_URL y TARGET_URL son variables de entorno inyectadas por docker-compose

Este Dockerfile va en la ruta: infra/jenkins_config/smee/Dockerfile

Dame el Dockerfile completo. Explica también:
- Qué es smee.io y por qué se necesita (Jenkins no tiene IP pública)
- Cómo verificar que el relay funciona (logs del contenedor y "Recent Deliveries" en GitHub)
```

---

### PROMPT 7 — Jenkinsfile Declarativo completo (Fase 4)

```
Soy estudiante de ingeniería de software. Necesito completar un Jenkinsfile declarativo
con 4 etapas para el proyecto StudyTrack.

Contexto:
- Backend: Spring Boot 3 / Java 17 / Maven en ruta "codigo_base/backend"
- Frontend: React 18 / Vite / TypeScript en ruta "codigo_base/frontend"
- Nexus Docker Registry: localhost:5000
- Nexus Maven Repo URL: http://nexus:8081/repository/maven-releases/
  (se usa "nexus" como hostname porque Jenkins y Nexus están en la misma red Docker)
- Credenciales de Nexus en Jenkins con ID: "nexus-credentials"
- El tag de imagen DEBE ser los primeros 7 chars del SHA del commit (git rev-parse --short=7 HEAD)
- NUNCA usar el tag :latest (política de empaquetamiento inmutable)
- Jenkins corre en Docker con acceso a /var/run/docker.sock

Tengo este esqueleto incompleto:

pipeline {
    agent any
    environment {
        IMAGE_TAG            = "TODO"
        NEXUS_REGISTRY       = "TODO"
        NEXUS_MAVEN_REPO     = "TODO"
        NEXUS_CREDENTIALS_ID = "TODO"
    }
    stages {
        stage('Checkout & Test') {
            steps {
                checkout scm
                script {
                    // TODO: calcular IMAGE_TAG y asignarlo a env.IMAGE_TAG
                }
                dir('codigo_base/backend') {
                    // TODO: mvn test
                }
            }
        }
        stage('Package & Tag Inmutable') {
            steps {
                dir('codigo_base/backend') {
                    // TODO: docker build con tag NEXUS_REGISTRY/studytrack-api:IMAGE_TAG
                }
                dir('codigo_base/frontend') {
                    // TODO: docker build con tag NEXUS_REGISTRY/studytrack-frontend:IMAGE_TAG
                }
            }
        }
        stage('Publish to Nexus') {
            steps {
                // TODO: withCredentials { docker login, docker push x2, docker logout }
                // TODO: withCredentials { mvn deploy del JAR }
            }
        }
        stage('Deploy & Smoke Test') {
            steps {
                dir('codigo_base/deploy') {
                    // TODO: docker compose up con IMAGE_TAG, sleep, curl smoke test
                }
            }
        }
    }
    post {
        success { echo "Pipeline finalizado en verde. Tag: ${env.IMAGE_TAG}" }
        failure { echo "El pipeline falló. Revise los logs." }
    }
}

Completa el Jenkinsfile con todos los TODOs. Usa withCredentials para inyectar
usuario/contraseña de Nexus. El smoke test debe hacer curl a http://localhost:8082/tasks
y fallar si no responde 200. Explica cada etapa.
```

---

### PROMPT 8 — deploy/docker-compose.yml (Fase 4)

```
Necesito completar el docker-compose.yml de despliegue de StudyTrack.
Este archivo es ejecutado por Jenkins en la etapa "Deploy & Smoke Test".

Contexto:
- Backend: imagen en localhost:5000/studytrack-api:${IMAGE_TAG}, puerto interno 8080
- Frontend: imagen en localhost:5000/studytrack-frontend:${IMAGE_TAG}, puerto interno 3000
- El frontend NO debe arrancar hasta que el backend esté HEALTHY
- El backend tiene endpoint /tasks que retorna 200 cuando está listo
- Ambos en la misma red Docker "product-network" con driver bridge
- Backend mapeado al puerto 8082 del host, frontend al 3000

Tengo este esqueleto:

version: "3.9"
services:
  backend:
    # TODO
    container_name:
    ports:
    networks:
    healthcheck:
      test:
      interval:
      timeout:
      retries:
      start_period:
  frontend:
    # TODO:
    image:
    container_name:
    ports:
    networks:
    depends_on:
      backend:
        condition:
networks:
  product-network:
    driver:

Complétalo. Explica la diferencia entre depends_on con condition: service_started
vs service_healthy, y por qué aquí se usa service_healthy.
```

---

### PROMPT 9 — Configurar Nexus: repositorios + usuario (Fase 3)

```
Tengo Nexus OSS 3 corriendo en http://localhost:8081. Acabo de levantarlo con Docker.
Necesito configurarlo para mi proyecto CI/CD.

Dame los pasos EXACTOS (con rutas de menú de la UI) para:

1. Primer login y cambio de contraseña:
   - Comando docker exec para obtener la contraseña inicial
   - Navegar a la UI y cambiar la contraseña
   - Deshabilitar el acceso anónimo en el wizard

2. Crear repositorio Docker hosted:
   - Nombre: docker-hosted
   - HTTP port: 5000
   - Deshabilitar anonymous pull

3. Crear repositorio Maven hosted (releases):
   - Nombre: maven-releases
   - Version policy: Release
   - Deployment policy: Allow redeploy

4. Crear usuario para Jenkins:
   - Username: jenkins-user
   - Rol: nx-admin

5. Verificar que todo funciona:
   - docker login localhost:5000
   - docker push de una imagen de prueba
   - curl a la API REST de Nexus para listar repositorios

Nota: Docker daemon ya está configurado con "insecure-registries": ["localhost:5000"]
en /etc/docker/daemon.json del host.
```

---

### PROMPT 10 — Configurar Jenkins: Credenciales + Pipeline + Webhook (Fase 3-4)

```
Tengo Jenkins corriendo en http://localhost:8080. Necesito configurarlo para el taller CI/CD.

Dame los pasos EXACTOS (con rutas de menú de la UI) para:

1. Setup inicial de Jenkins:
   - Comando docker exec para obtener la contraseña inicial
   - Desbloquear Jenkins y instalar plugins sugeridos
   - Crear el usuario administrador

2. Agregar credencial de Nexus:
   - Ruta: Manage Jenkins > Credentials > System > Global credentials > Add Credentials
   - Kind: Username with password
   - ID exacto: "nexus-credentials"
   - Username: jenkins-user / Password: (el que creaste en Nexus)

3. Crear el Pipeline:
   - New Item > Pipeline
   - Nombre: studytrack-pipeline
   - Definition: Pipeline script from SCM
   - SCM: Git, URL del repositorio de GitHub
   - Branch: */main
   - Script Path: codigo_base/Jenkinsfile
   - Activar: GitHub hook trigger for GITScm polling

4. Ejecutar el primer build manualmente y verificar cada etapa

5. Hacer git push para verificar que el trigger automático funciona

Dame también los errores más comunes en cada paso y cómo resolverlos.
```

---

### PROMPT 11 — Troubleshooting completo del ecosistema

```
Estoy haciendo un taller de CI/CD con Jenkins + Nexus + Docker en Ubuntu local.
Necesito una guía de troubleshooting para los errores más comunes.

Dame soluciones para:

1. "Got permission denied while trying to connect to the Docker daemon socket"
   - Jenkins no puede usar el socket de Docker
   - Cómo verificar y corregir el GID del grupo docker

2. "http: server gave HTTP response to HTTPS client"
   - Jenkins no puede hacer docker push al registry de Nexus (HTTP, no HTTPS)
   - Cómo configurar insecure-registries en /etc/docker/daemon.json

3. "401 Unauthorized" al hacer mvn deploy a Nexus
   - Credenciales incorrectas o permisos faltantes en Nexus

4. El webhook de GitHub no dispara el pipeline
   - Cómo verificar los logs de smee-client
   - Cómo verificar "Recent Deliveries" en GitHub
   - Cómo verificar que Jenkins tiene el trigger activado

5. Nexus no arranca (healthcheck falla)
   - Nexus tarda hasta 5 minutos en iniciar
   - Cómo ver los logs y qué línea indica que está listo

6. Puerto ya en uso al levantar los contenedores
   - Cómo identificar qué proceso usa el puerto
   - Cómo cambiar el puerto en el .env

Para cada caso: síntoma exacto, causa raíz y solución paso a paso.
```

---

## 🔧 Troubleshooting frecuente

### Docker socket: permiso denegado

```bash
# Verificar el GID real del socket en el host
ls -la /var/run/docker.sock
# Si el GID es diferente a 999, ajustar el Dockerfile de Jenkins:
# RUN groupadd -f -g <GID_REAL> docker && usermod -aG docker jenkins

# Reiniciar el stack de Jenkins después del cambio
cd infra/jenkins_config
docker compose down
docker compose up -d --build
```

### Nexus registry inseguro (HTTP sin TLS)

```bash
# En el HOST de Ubuntu:
sudo nano /etc/docker/daemon.json
```
```json
{
  "insecure-registries": ["localhost:5000"]
}
```
```bash
sudo systemctl restart docker
docker info | grep -A3 "Insecure Registries"
```

### Puerto ya en uso

```bash
sudo lsof -i :8080   # Jenkins
sudo lsof -i :8081   # Nexus UI
sudo lsof -i :5000   # Nexus Docker Registry
# Cambiar el puerto en el .env correspondiente
```

### Nexus tarda en arrancar

```bash
# Esperar hasta 5 minutos — es normal
docker logs nexus --follow
# Buscar la línea: "Started Sonatype Nexus OSS"
```

### mvn deploy falla con 401 Unauthorized

- Verificar que el ID de credencial en Jenkins es exactamente `nexus-credentials`
- Verificar que el usuario de Nexus tiene rol `nx-admin`
- Verificar que el repositorio `maven-releases` tiene **Deployment policy: Allow redeploy**

### Pipeline no se dispara con git push

```bash
# 1. Verificar que smee-client está corriendo y conectado
docker logs smee-client --follow

# 2. Verificar en GitHub: Settings > Webhooks > tu webhook > Recent Deliveries
# Debe haber entregas recientes con status 200

# 3. En Jenkins: verificar que el pipeline tiene "GitHub hook trigger" activado
```

---

## ✅ Checklist final de entrega

### Fase 1
- [ ] `mvn test` pasa con `BUILD SUCCESS`
- [ ] `npm run build` genera la carpeta `dist/`

### Fase 2
- [ ] `backend/Dockerfile` tiene 2 stages (builder JDK → runtime JRE)
- [ ] `frontend/Dockerfile` tiene 2 stages (builder Node → runtime Nginx)
- [ ] `frontend/nginx.conf` tiene `try_files` para SPA fallback
- [ ] `backend/pom.xml` tiene bloque `<distributionManagement>` con URL usando hostname `nexus`
- [ ] La versión del pom es fija (no SNAPSHOT)
- [ ] `docker build` del backend genera imagen < 300MB
- [ ] `docker build` del frontend genera imagen < 50MB

### Fase 3
- [ ] Red `cicd_network` creada
- [ ] Nexus levanta y responde en `http://localhost:8081`
- [ ] Nexus tiene repositorio `docker-hosted` en puerto 5000
- [ ] Nexus tiene repositorio `maven-releases`
- [ ] `/etc/docker/daemon.json` tiene `insecure-registries: ["localhost:5000"]`
- [ ] `docker login localhost:5000` funciona con el usuario de Nexus
- [ ] Jenkins levanta y responde en `http://localhost:8080`
- [ ] Jenkins tiene credencial con ID `nexus-credentials`
- [ ] Canal de smee.io creado y configurado en GitHub webhook
- [ ] `smee-client` corre y los logs muestran conexión activa

### Fase 4
- [ ] `deploy/docker-compose.yml` completo con healthcheck y `service_healthy`
- [ ] `Jenkinsfile` completo con las 4 etapas
- [ ] Pipeline ejecuta manualmente sin errores
- [ ] `git push` dispara el pipeline automáticamente
- [ ] Backend accesible en `http://localhost:8082/tasks`
- [ ] Frontend accesible en `http://localhost:3000`
- [ ] Las imágenes en Nexus tienen tag SHA de 7 caracteres (no `:latest`)
- [ ] El JAR aparece en `maven-releases` de Nexus UI

---

## 📚 Referencias rápidas

| Recurso                        | URL                                                        |
|--------------------------------|------------------------------------------------------------|
| Jenkins Pipeline Syntax        | https://www.jenkins.io/doc/book/pipeline/syntax/           |
| withCredentials (Jenkins)      | https://www.jenkins.io/doc/pipeline/steps/credentials-binding/ |
| Nexus REST API                 | http://localhost:8081/service/rest/v1/repositories         |
| Smee.io                        | https://smee.io                                            |
| Docker multi-stage builds      | https://docs.docker.com/build/building/multi-stage/        |
| eclipse-temurin images         | https://hub.docker.com/_/eclipse-temurin                   |

---

*Guía generada para el Taller Evaluativo 2 — Ingeniería de Software V — Universidad Icesi — Período 202620*
