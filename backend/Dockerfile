FROM golang:1.26-alpine3.23 AS builder
WORKDIR /app

COPY go.mod go.sum ./
RUN go mod download

COPY . .

RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /app/main ./cmd/api

FROM alpine:3.23
WORKDIR /app

RUN addgroup -g 1000 app \
    && adduser -G app -u 1000 app -D

COPY --from=builder --chown=app:app /app/main /app/main

USER app:app

EXPOSE 8081

CMD ["/app/main"]
