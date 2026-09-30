# =============================================================================
# Taller Evaluativo 2 - Fase 2: Empaquetamiento Inmutable Multi-Stage
# Frontend: studytrack-frontend
# =============================================================================

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
