# =============================================================================
# Taller Evaluativo 2 - Fase 2: Empaquetamiento Inmutable Multi-Stage
# Backend: studytrack-api
#
# Complete los TODOs siguiendo las buenas prácticas vistas en el curso:
# reducción de peso, cache de dependencias, usuario no root y ausencia
# total del SDK/compilador en la imagen final de runtime.
# =============================================================================

# ---------- Stage 1: Builder ----------
FROM maven:3.9-eclipse-temurin-17-alpine AS builder

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