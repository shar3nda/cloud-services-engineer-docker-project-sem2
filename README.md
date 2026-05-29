# Крупнов Иван Игоревич - Проект 2 семестра

## Архитектура

Перед приложением стоит гейтвей (балансировщик) в виде nginx (в режиме реверс-прокси). Он проксирует запросы к фронтенду и бэкенду, в зависимости от запрошенного URL. Балансировку между репликами фронтенда и бэкенда выполняет сам Docker. В изолированных сетях за балансировщиком стоят фронтенд (nginx в режиме веб-сервера) и бэкенд (go-микросервис).

## Запуск

В продовом режиме:

```bash
docker compose -f docker-compose.yml up -d
```

В дев-режиме:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
```

## Масштабирование

По умолчанию бэкенд и фронтенд запускаются в двух репликах. Также возможно ручное масштабирование:

```bash
docker compose -f docker-compose.yml up -d --scale backend=3 --scale frontend=3
```

Гейтвей не масштабируется, тк он публикует порт `80` наружу.

## Конфигурация

Порт бэкенда: `8081`
Порт гейтвея и фронтенда: `80`
Переменные сборки Docker:
- `VUE_APP_BACKEND_URL` - URL бэкенда (по умолчанию `/api`)
- `VUE_APP_PUBLIC_PATH` - базовый путь для фронтенда (по умолчанию `/`)

## Безопасность

- Используются легкие базовые образы на основе alpine
- Контейнеры запускаются от не-root пользователя
- Ограничены linux-капабилити контейнеров: `cap_drop: ALL`, `cap_add: NET_BIND_SERVICE` для 80 порта; `security_opt: no-new-privileges` для отключения эскалации привилегий
- В пайплайн встроено сканирование образов на уязвимости с помощью Trivy; при нахождении CRIT/HIGH уязвимостей сборка прерывается и образы не публикуются в Docker Hub
- Везде используется readonly файловая система, с исключением в виде tmp-директорий и томов Docker для логов nginx
- Сети фронтенда и бэкенда изолированы
- В workflow `aquasecurity/trivy-action` запинен по хэшу коммита в связи с недавним инцидентом безопасности этого экшена
- `DOCKER_USER`, `DOCKER_PASSWORD` хранятся в виде секретов GitHub Actions

## Образы

В бэкенде и фронтенде используется multi-stage сборка и оптимизации размера.

### Бэкенд

- Используется multi-stage сборка, итоговый образ - alpine с единственным бинарником приложения
- Оптимизирован размер бинарника: `CGO_ENABLED=0 GOOS=linux go build -trimpath -ldflags="-s -w" -o /app/main ./cmd/api`
- Настроено кэширование зависимостей в отдельном слое: `COPY go.mod go.sum ./`, `RUN go mod download`

### Фронтенд

- Используется multi-stage сборка, итоговый образ - nginx-unprivileged (безопасный nonroot-образ) с собранными статическими файлами
- Используется `npm ci` для вызова ошибки при несовпадении `package-lock.json` и `package.json`
- Настроено кэширование зависимостей в отдельном слое: `COPY package*.json ./`, `RUN npm ci`

### Размер образов

```bash
$ docker image ls --format table | grep docker-project
johannkrupp/docker-project-frontend       latest                          db84eadebea0   4 minutes ago   83.7MB
johannkrupp/docker-project-backend        latest                          41027999fa38   4 minutes ago   28.6MB
```

## Healthchecks

Настроены healthchecks для всех сервисов:
- Бэкенд: `["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8081/health"]`
- Гейтвей/фронтенд: `["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost/"]`

### Примечания

- Не используются секреты docker, тк в этом приложении нет секретов (однако используются секреты github actions для юзера и токена Docker Hub)
- Не используются profiles в docker compose, вместо них - override-файл docker-compose.dev.yml для дев-режима
- Для healthcheck используется wget, тк он является частью busybox в alpine, а curl там не предустановлен и его незачем тащить в образ.
