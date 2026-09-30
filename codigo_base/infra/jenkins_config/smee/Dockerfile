# Contenedor liviano que ejecuta smee-client: reenvía los eventos del
# webhook de GitHub (recibidos en un canal público de smee.io) hacia el
# endpoint /github-webhook/ de Jenkins, sin necesidad de exponer Jenkins
# a Internet.
FROM node:20-alpine

RUN npm install -g smee-client
ENTRYPOINT ["sh", "-c", "smee --url \"$SMEE_URL\" --target \"$TARGET_URL\""]