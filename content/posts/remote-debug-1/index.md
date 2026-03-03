---
title: Remote Debugging for Vulnerability Research (Part 1. Go lang)
description: Debugging is an important part of the vulnerability research process
date: 2025-03-01T00:00:00+01:00
tldr: 
draft: false
tags: [web]
toc: false
---

Всем привет! В процессе vulnerability research-а в формате white box нам постоянно приходится не просто читать код, а понимать что происходит во время исполнения программы. И тут без дебаггера никуда. Причём дебажить приходится далеко не всегда локально. Многие (и я в том числе) по возможности стараются поднимать тестируемые приложения в Docker’e или отдельной виртуальной машине — это даёт изоляцию, гибкость и возможность быстро деплоить окружение. Но именно из-за такого подхода локальный дебаггинг становится невозможным. Поэтому рано или поздно всё сводится к необходимости подключаться к процессу удалённо и разбираться с ним прямо там, где он работает.

> [!To be calm]
> Тема, конечно, далеко не новая. Разработчики, OSWE-ребята и просто white-box-энтузиасты — не ругайтесь: да, многие из вас делают это годами. Но тем не менее, иногда полезно взглянуть на привычные вещи под другим углом и собрать в одном месте то, что обычно раскидано по заметкам и личному опыту.

{{< callout emoji="⚡️" text="Original callout." >}}

В этом цикле статей я постараюсь раскрыть, как использовать удалённый дебаггинг именно в контексте vuln research-а: какие инструменты подходят для разных окружений, что важно учитывать и как сделать этот процесс максимально удобным и эффективным.

`ну а получится или нет это другой вопрос :)`

Я планирую сделать серию статей и показать процесс для разных языков, как компилируемых так и для интерпретируемых. Сегодня начнем с go-шки на примере достаточно большого проекта - [gitea](https://github.com/go-gitea/gitea)

## Initial deployment

Начнем с изучения документации по способам деплоя gitea. Быстро находим простейший вариант для [деплоя при помощи Docker](https://docs.gitea.com/installation/install-with-docker). Отлично! Можем забрать уже созданный docker-compose файлик и быстренько поднять сервис, как базу будем использовать postgres.

`docker-compose.yaml` - дефолтный (для теста)

```yaml
version: "3"

networks:
  gitea:
    external: false

services:
  server:
    image: docker.gitea.com/gitea:1.25.4
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
+      - GITEA__database__DB_TYPE=postgres
+      - GITEA__database__HOST=db:5432
+      - GITEA__database__NAME=gitea
+      - GITEA__database__USER=gitea
+      - GITEA__database__PASSWD=gitea
    restart: always
    networks:
      - gitea
    volumes:
      - ./gitea:/data
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "3000:3000"
      - "222:22"
+    depends_on:
+      - db
+
+  db:
+    image: docker.io/library/postgres:14
+    restart: always
+    environment:
+      - POSTGRES_USER=gitea
+      - POSTGRES_PASSWORD=gitea
+      - POSTGRES_DB=gitea
+    networks:
+      - gitea
+    volumes:
+      - ./postgres:/var/lib/postgresql/data
```

Теперь можем поиграться с функциональностью сервиса, но, само собой, при такой конфигурации ни о каком remote-debug'e и речи идти не может.
## Remote Debugging Configuration

Что же нам ==необходимо для удалённого дебага Gitea==:
1. Так как go-шка - компилируемый язык, первым делом нам нужно правильно собрать бинарь, чтобы сохранить debug-информацию. 
2. Поскольку в контейнере будет крутиться просто бинарь (без рантайма, как у интерпретируемых языков, где можно включить debug средствами конфигурации), нам понадобится сторонний дебаггер. Мы будем использовать [Delve](https://github.com/go-delve/delve).
3. Нужно будет написать новый Dockerfile, так как в compose используется готовый образ из Docker Registry, который нам не подойдёт, и поправить сам docker- compose.yaml.
4. Ну и настроить vs-code для подключения к удаленному процессу.

Ну что, начинаем! 

### 1. Correct build of the binary 

При удалённой отладке Go-сервиса важно правильно собрать бинарь. По умолчанию компилятор Go активно оптимизирует код: переупорядочивает инструкции, удаляет временные переменные и встраивает функции (inline). Это ускоряет программу, но сильно осложняет работу дебаггера - c брейкпоинтами что-то не так, стек вызовов становится нечитаемым, а переменные могут быть недоступны.

Чтобы сохранить корректную отладочную информацию, бинарь нужно собрать **без оптимизаций и inline-встраивания функций**:
```bash
go build -gcflags="all=-N -l" -o gitea-debug .
```

Подробнее по флагам:
- **`-N`** — отключает оптимизации компилятора.  
    Благодаря этому строки исходного кода соответствуют инструкциям в бинаре, и переменные не удаляются компилятором.
- **`-l`** — отключает inline функций.  
    Это важно для корректного отображения стека вызовов и возможности ставить breakpoints внутри функций.
- **`all=`** — применяет флаги ко всем пакетам, включая зависимости.

### 2. Delve debugger  

Для удалённой отладки Go-приложений стандартным инструментом является **Delve** - это официальный дебаггер для Go-шки. 

Delve можно использовать в двух режимах:
1. **Запуск процесса под управлением Delve**
2. **Подключение к уже запущенному процессу**

Мы будем использовать первый вариант запускать бинарь под управлением delve.

```bash
dlv exec ./app --headless --listen=:40000 --api-version=2
```

Что же происходит под капотом когда мы запускаем эту команду?
1. Delve стартует как управляющий процесс. Он не просто запускает бинарь, а запускает его через механизм отладки ОС (ptrace на Linux). Это позволяет Delve: останавливать выполнение программы, читать память процесса, менять значения переменных, управлять горутинами и так далее
2. Далее Delve запускает наш бинарь, как дочерний процесс
3. Headless-режим открывает debug-сервер

	Флаг:
	```bash
	--headless
	```
	
	говорит Delve не открывать интерактивную консоль, а запустить сервер.
	
	Флаг:
	```bash
	--listen=:40000
	```
	
	открывает TCP-порт. На этот порт чуть позже будем подключатся при помощи IDE. По сути Delve поднимает небольшой RPC-сервер.

### 3.  Debug docker file  

Окей переходим к следующему этапу! Мы поняли как собрать бинарь правильно и как его запустить, что бы можно было прицепится к процессу дебага удаленно. Теперь надо обернуть это все в Dockerfile и собственно можно приступать к отладке. 

Разворачивать debug версию будем в два этапа: первый стейдж - сборка исполняемого файла, далее уже поднимаем его под дебагером. 

Dockerfile.debug:

```dockerfile
# ---- build stage ----
FROM golang:1.25.1 AS builder
WORKDIR /app
RUN apt-get update && apt-get install -y curl gnupg make git && \
curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && \
apt-get install -y nodejs && npm install -g yarn && \
apt-get clean && rm -rf /var/lib/apt/lists/*
RUN git clone https://github.com/go-gitea/gitea.git . --branch v1.24.5
RUN go install github.com/go-delve/delve/cmd/dlv@latest
RUN make frontend && make generate
# См. пункт 1
RUN go build -gcflags="all=-N -l" -o /usr/local/bin/gitea-debug .

# ---- deploy stage ----
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates git && \
rm -rf /var/lib/apt/lists/*
WORKDIR /app
RUN useradd -m -u 1000 gitea && \
mkdir -p /app && chown -R gitea:gitea /app
COPY --from=builder /go/bin/dlv /usr/local/bin/dlv
COPY --from=builder /usr/local/bin/gitea-debug /usr/local/bin/gitea-debug
COPY --from=builder /app /app
RUN chown -R gitea:gitea /app
USER gitea
EXPOSE 3000 40000
ENV GITEA_WORK_DIR=/app
ENV GITEA_CUSTOM=/app/custom
# См. пункт 2
ENTRYPOINT ["dlv", "exec", "/usr/local/bin/gitea-debug","--headless", "--listen=:40000", "--api-version=2", "--accept-multiclient","--", "web", "--config", "/app/custom/conf/app.ini", "--work-path", "/app"]
```

#### Build stage

Берём официальный образ Go, ставим зависимости для сборки фронтенда и генерации ресурсов 

```dockerfile
FROM golang:1.25.1 AS builder  
WORKDIR /app
RUN apt-get update && apt-get install -y curl gnupg make git && \  
  curl -fsSL https://deb.nodesource.com/setup_20.x | bash - && \  
  apt-get install -y nodejs && npm install -g yarn && \  
  apt-get clean && rm -rf /var/lib/apt/lists/*
```

Клонируем исходники нужной нам версии gitea (я смотрел 1.24.5 версию):
```dockerfile
RUN git clone https://github.com/go-gitea/gitea.git . --branch v1.24.5
```

Ставим Delve:
```dockerfile
RUN go install github.com/go-delve/delve/cmd/dlv@latest
```

Так мы получаем бинарь `dlv` в `/go/bin/dlv`, который потом перенесём в финальный образ.  

Собираем фронтенд и генерируем ресурсы:
```dockerfile
RUN make frontend && make generate
```

Ну и билдим сам gitea ([[#1. Correct build of the binary|См. пункт 1]]): 
```dockerfile
RUN go build -gcflags="all=-N -l" -o /usr/local/bin/gitea-debug .
```

#### Deploy stage

В финальном стейдже используем легкий образ, ставим необходимые зависимости и создаем пользователя:
```dockerfile
FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y ca-certificates git && \  
  rm -rf /var/lib/apt/lists/*
RUN useradd -m -u 1000 gitea && \  
  mkdir -p /app && chown -R gitea:gitea /app
```

Копируем артефакты сборки:
```dockerfile
COPY --from=builder /go/bin/dlv /usr/local/bin/dlv  
COPY --from=builder /usr/local/bin/gitea-debug /usr/local/bin/gitea-debug  
COPY --from=builder /app /app
```

Здесь важный момент: мы копируем не только бинарь, но и `/app`, потому что там лежит рабочая директория проекта, конфиги/ресурсы/статика.

Меняем владельца директории с gitea и переключаемся на пользователя:
```dockerfile
RUN chown -R gitea:gitea /app  
USER gitea
```

Открываем порты:
```dockerfile
EXPOSE 3000 40000
```
- `3000` — веб-интерфейс Gitea
- `40000` — Delve debug port

И задаём переменные окружения:
```dockerfile
ENV GITEA_WORK_DIR=/app  
ENV GITEA_CUSTOM=/app/custom
```
Это говорит Gitea, где искать рабочую директорию и кастомные файлы, конфиги.

Ну и в конце мы запускаем бинарник из под delve ([[#2. Delve debugger|См. пункт 2]])
```dockerfile
ENTRYPOINT ["dlv", "exec", "/usr/local/bin/gitea-debug",  
  "--headless",  
  "--listen=:40000",  
  "--api-version=2",  
  "--accept-multiclient",  
  "--",  
  "web", "--config", "/app/custom/conf/app.ini", "--work-path", "/app"]
```

Важно учитывать, что все что идет после `--` - это аргументы уже для Gitea, а не для Delve. Запускаем gitea в режиме web. (Более подробное описание аргументов можно найти в [документации](https://docs.gitea.com/administration/command-line))

#### Docker compose 

Нужно обновить `docker-compose.yaml`, чтобы контейнер запускался именно из нашего `Dockerfile.debug`, открывал debug-порт и имел нужные права для работы Delve.

Достаточно обновить просто server секцию

```yaml
server:
  build:
    context: .
    dockerfile: Dockerfile.debug
  container_name: gitea-debug

  environment:
    - USER_UID=1000
    - USER_GID=1000
    - GITEA__database__DB_TYPE=postgres
    - GITEA__database__HOST=db:5432
    - GITEA__database__NAME=gitea
    - GITEA__database__USER=gitea
    - GITEA__database__PASSWD=gitea
  networks:
    - gitea
  volumes:
    - ./gitea:/data
    - ./conf/app.ini:/app/custom/conf/app.ini
  ports:
    - "3000:3000"
    - "40000:40000"
  cap_add:
    - SYS_PTRACE
  security_opt:
    - seccomp=unconfined
  depends_on:
    - db
```

Тут также все стандартно. Меняем image секцию на build, чтобы указать что собираем образ из нашего Dockerfile.debug. Пробрасываем необходимые нам порты. И на всякий случай добавим нужные capabilities и отключим Secure Computing Mode 

### 4.  VScode attach

Почти все! На этом этапе у нас уже есть контейнер, внутри которого Gitea запущена под Delve headless и слушает порт `40000`. Осталось подключить к этому процессу при помощи VS Code и найти пару 0day-ев :).  

> [!Важное условие]
> Локальные исходники должны совпадать с исходниками в контейнере 
>  

По этому в нашем случае локально стягиваем именно версию 1.24.5
```bash
git clone https://github.com/go-gitea/gitea.git
cd gitea
git checkout v1.24.5
```

Осталось только написать конфигурационный файл для vscode. 

.vscode/launch.json
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Attach to Gitea",
      "type": "go",
      "request": "attach",
      "mode": "remote",
      "host": "127.0.0.1",
      "port": 40000,
      "apiVersion": 2,
      "showLog": true,
      "trace": "verbose",
      "debugAdapter": "dlv-dap",
      "substitutePath": [
        { "from": "/Users/hx0110/VulnResearch/gitea/infra/debug", "to": "/app" }
      ]
    }
  ]
}
```

Важные моменты:    
1. **`request: "attach"`**  
    Указываем, что мы будем подключаться к существующему процессу
2. **`mode: "remote"`**  
    При этом говорим, что процесс находится удаленно. Не забываем указать host и port который слушает delve (`host:127.0.0.1` и `port:40000`)    
3. **`debugAdapter: "dlv-dap"`**  
    Использовать DAP-адаптер (Debug Adapter Protocol). Обычно это самый стабильный вариант
4. **`substitutePath`**  
    Самый важный пункт для Docker/remote debug: сопоставляет пути файлов в контейнере и на твоей машине
    - `to: "/app"` — путь к исходникам внутри контейнера
    - `from: "/Users/.../infra/debug"` — путь к тем же исходникам у тебя локально
## Test time

Отлично! Все готово. Теперь нам остается просто запустить docker-compose.yaml, прицепиться к процессу и проверить, что все работает.

![demo](images/demo.gif)

Ну что, всем спасибо! На этом первую часть я думаю закончим, буду очень признателен фитбеку (можно оставить в [телеге](https://t.me/hx0110)). Как уже писал выше, план - сделать цикл статей по удаленной отладке в целом для разных языков и может по каким-то особеностям именно для Vulnerability Research-а. Еще раз все спасибо!)
