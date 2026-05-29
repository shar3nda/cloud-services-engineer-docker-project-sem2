FROM node:26-alpine3.23 as builder
WORKDIR /app

ENV NODE_OPTIONS="--openssl-legacy-provider"

ARG VUE_APP_API_URL="/api"
ENV VUE_APP_API_URL=$VUE_APP_API_URL
ARG VUE_APP_PUBLIC_PATH="/"
ENV VUE_APP_PUBLIC_PATH=$VUE_APP_PUBLIC_PATH

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

FROM nginxinc/nginx-unprivileged:1.31-alpine3.23
WORKDIR /app

COPY --from=builder --chown=nginx:nginx /app/dist /usr/share/nginx/html
COPY --chown=nginx:nginx nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
