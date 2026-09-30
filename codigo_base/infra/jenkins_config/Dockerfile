# Imagen de Jenkins con el CLI de Docker, Docker Compose y Maven instalados, para que el propio
# controlador (montando /var/run/docker.sock) pueda construir y publicar
# las imágenes del backend y del frontend durante el pipeline.
FROM jenkins/jenkins:lts-jdk17

USER root

# Instalar Docker CLI, plugin de compose y Maven
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
    apt-get install -y docker-ce-cli docker-compose-plugin maven && \
    rm -rf /var/lib/apt/lists/*

# Agregar jenkins al grupo docker del host (GID 999 es el habitual en Ubuntu)
RUN groupadd -f -g 115 docker && usermod -aG docker jenkins

COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

USER jenkins