В Docker есть два движка для `docker build`:
- Старый (legacy builder) — был всегда.
-  Новый (BuildKit) — появился в 2018

В современном Docker Desktop BuildKit включён по умолчанию. На Linux:

`DOCKER_BUILDKIT=1 docker build .`

Или в `~/.docker/config.json`:

```json
{ "features": { "buildkit": true } }
```

BuildKit умеет несколько штук, которых не было у старого:

# Параллельные стадии
Старый движок собирал стадии по очереди: пока первая не закончилась, вторая ждёт. BuildKit строит граф зависимостей и параллелит. Если стадия A не зависит от стадии B — они собираются одновременно.

# Постоянный кэш для зависимостей
В обычной сборке `go mod download` или `npm install` повторяется при каждом изменении `go.mod` / `package.json`. Качаются мегабайты заново. С BuildKit можно сказать: «эту директорию кэшируй между сборками». При следующей сборке зависимости берутся из локального кэша.

Самая медленная операция в сборке Go-приложения — `go mod download`. Тянет десятки модулей из интернета. Обычный кэш слоёв спасает только пока `go.mod` не меняется. Добавил одну зависимость → весь `go mod download` пересобирается. Решение BuildKit: Можно сказать: «храни кэш `go/pkg/mod` между сборками отдельно от слоёв».

```
# syntax=docker/dockerfile:1.7
FROM golang:1.26

WORKDIR /src
COPY go.mod go.sum ./

RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .
RUN --mount=type=cache,target=/go/pkg/mod \
    CGO_ENABLED=0 go build -o /app
```

`# syntax=docker/dockerfile:1.7`: Это говорит BuildKit использовать новый синтаксис с поддержкой `--mount`. Без этой строки команда не сработает.

Что делает `--mount=type=cache,target=/go/pkg/mod`:  
- Монтирует постоянное хранилище в `/go/pkg/mod` (там Go хранит модули).  
- Этот кэш не попадает в образ.  
- Но сохраняется между сборками.

Результат:  
- Первая сборка: тянет всё с нуля.  
- Добавили зависимость: качается только новый модуль.  
- Изменили код: ничего не качается.

# Секреты без записи в слой
Если нужен токен (например для `git clone` приватного репо), в старом подходе вы передавали его через `ARG` или `env`. Беда — токен оставался в истории образа. Кто получил образ → достал токен через `docker history`. BuildKit монтирует секрет как файл только на момент команды. После — никаких следов.

```
# syntax=docker/dockerfile:1.7
FROM golang:1.26

RUN --mount=type=secret,id=github_token \
    GH_TOKEN=$(cat /run/secrets/github_token) && \
    git clone https://$GH_TOKEN@github.com/private/repo.git
```

Сборка:

```bash
docker build --secret id=github_token,src=$HOME/.gh_token .
```

Что происходит:  
- Файл `~/.gh_token` монтируется как `/run/secrets/github_token` внутри образа.  
- Только на время выполнения этого `RUN`.  
- В слой не попадает. В историю не записывается.