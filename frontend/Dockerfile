FROM node:26-alpine3.23 as builder
WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

FROM nginx:1.31-alpine3.23
WORKDIR /app

RUN addgroup -g 1000 app \
    && adduser -G app -u 1000 app -D

COPY --from=builder --chown=app:app /app/dist /usr/share/nginx/html
COPY --chown=app:app nginx.conf /etc/nginx/conf.d/default.conf

USER app:app

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
