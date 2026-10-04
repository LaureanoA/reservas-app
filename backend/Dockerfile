FROM node:20-alpine AS deps

WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci --omit-dev




FROM node:20-alpine

WORKDIR /app

ENV NODE_ENV=production

ARG APP_VERSION=v1
ENV APP_VERSION=${APP_VERSION}

COPY --from=deps --chown=node:node /app/node_modules ./node_modules
COPY --chown=node:node package.json package-lock.json ./
COPY --chown=node:node server.js db.js ./

USER node 

EXPOSE 3000

CMD ["node", "server.js"]
