FROM node:24.21.0-alpine

ENV NODE_ENV=production
WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force

COPY src ./src

EXPOSE 3000

CMD ["node", "src/server.js"]
