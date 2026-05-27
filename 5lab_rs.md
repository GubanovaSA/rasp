# Лабораторная работа 05

## Проектирование и реализация комплексной микросервисной системы для автоматизации бизнес-процесса

---

**Студент:** Губанова Светлана

**Вариант:** 6

---

## 1. Цель работы

### запускать многоконтейнерные приложения, организовывать взаимодействие между сервисами, использовать Docker Compose для оркестрац,изменять бизнес-логику и инфраструктуру проекта; , работать с Redis как с внешним сервисом хранения данных.
---

## 2. Окружение

| Компонент | Версия | Проверка |
|-----------|--------|----------|
| ОС | macOS Sequoia | — |
| Docker | 27.x | `docker --version` |
| Docker Compose | v2.x | `docker compose version` |
| Python | 3.9+ | `python3 --version` |

---

## 3. Архитектура решения

```
┌─────────────────────────────────────────────────┐
│                 ТВОЙ КОМПЬЮТЕР                   │
│                                                 │
│  Браузер ── localhost:8000 ──┐                  │
│                               ▼                  │
│  ┌──────────────────────────────────────────┐   │
│  │            DOCKER                        │   │
│  │                                          │   │
│  │  ┌────────────────┐   ┌───────────────┐  │   │
│  │  │   FRONTEND      │   │    REDIS      │  │   │
│  │  │   Flask :5000   │──►│   :6379       │  │   │
│  │  │   ENV TEAM=Sales│   │   ключ: hits  │  │   │
│  │  └───────┬─────────┘   └───────────────┘  │   │
│  │          │                                 │   │
│  │          ▼                                 │   │
│  │   logs/debug.log (JSON)                   │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**Как идёт запрос:**
1. Браузер → `localhost:8000` → Docker пробрасывает в контейнер на порт 5000
2. Flask вызывает `cache.incr('hits')` → Redis (`redis:6379`) увеличивает счётчик
3. Flask определяет сценарий (`get_decision`) и пишет JSON-лог
4. Flask возвращает HTML-страницу

**Ключевые термины:**

| Термин | Значение |
|--------|----------|
| **Контейнер** | Изолированная среда с приложением и всеми зависимостями |
| **Образ** | Шаблон для создания контейнера |
| **Docker Compose** | Запуск нескольких контейнеров одной командой |
| **Volume** | Проброс папки с компьютера в контейнер |
| **Flask** | Веб-фреймворк на Python |
| **Redis** | In-memory база данных (ключ → значение) |

---

## 4. Индивидуальное задание

| Задача | Файл | Что изменить |
|--------|------|--------------|
| 1 | `app.py` | Перевести интерфейс на английский |
| 2 | `docker-compose.yml` | Переименовать `web` → `frontend` |
| 3 | `Dockerfile` | Добавить `ENV TEAM="Sales"` |

---

## 5. Ход работы

### 5.1. Файлы проекта

```
lab5_business/
├── app.py               # Код приложения
├── Dockerfile            # Инструкция сборки образа
├── docker-compose.yml    # Конфигурация сервисов
├── requirements.txt      # Зависимости Python
└── logs/debug.log        # JSON-логи (создаётся сам)
```

### 5.2. requirements.txt (без изменений)

```txt
Flask==2.0.1
Werkzeug==2.3.7
redis==4.6.0
```

### 5.3. Dockerfile — Задача 3

**добавлен ENV:**
```dockerfile
FROM python:3.9-alpine

# Задача 3: переменная окружения
ENV TEAM="Sales"

WORKDIR /code
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

**Что такое ENV:** задаёт переменную окружения внутри контейнера. Любой процесс может её прочитать.

### 5.4. docker-compose.yml — Задача 2

**С изменением (переименован сервис):**
```yaml
services:
  # Задача 2: web → frontend
  frontend:
    build: .
    ports:
      - "8000:5000"
    depends_on:
      - redis
    volumes:
      - ./logs:/code/logs
  redis:
    image: redis:alpine
```

**Что изменилось:** имя сервиса `web` заменено на `frontend`. Контейнер теперь называется `lab5_business-frontend-1`. DNS-имя в сети Docker — `frontend`.

### 5.5. app.py — Задача 1

**Что изменено:** все русские надписи переведены на английский.

| Было | Стало |
|------|-------|
| `"Супер-приз!"` | `"Super Prize!"` |
| `"Юбилейный посетитель!"` | `"Milestone Visitor!"` |
| `"Счастливый посетитель!"` | `"Lucky Visitor!"` |
| `"Чётный посетитель"` | `"Even Visitor"` |
| `"Обычный посетитель"` | `"Regular Visitor"` |
| `"Вы посетитель номер:"` | `"Visitors count:"` |
| `"Логи сохраняются в:"` | `"Logs saved to:"` |
| `<html lang="ru">` | `<html lang="en">` |

### 5.6. Запуск

```bash
docker compose up -d --build
```
<img width="717" height="540" alt="image" src="https://github.com/user-attachments/assets/bbc1ae2a-b15c-43e8-9bd9-6f0bbfe70292" />


---

## 6. Проверка работоспособности

| Что проверяю | Команда | Ожидаемый результат |
|--------------|---------|---------------------|
| Контейнеры | `docker compose ps` | `frontend` и `redis` в статусе `Up` |
| Сайт | `http://localhost:8000` | Интерфейс на английском, счётчик растёт |
| Задача 3 | `docker compose exec frontend env \| grep TEAM` | `TEAM=Sales` |
| Redis | `docker compose exec redis redis-cli GET hits` | Показывает число |
| Логи | `docker compose exec frontend cat logs/debug.log` | JSON-записи |
| Healthcheck | `http://localhost:8000/health` | `{"status":"ok","redis":"connected"}` |

<img width="419" height="388" alt="image" src="https://github.com/user-attachments/assets/b7a642dc-a62e-4e29-a160-53b9030cd1e8" />

## логи <img width="635" height="401" alt="image" src="https://github.com/user-attachments/assets/4e4ad674-278e-4ee9-985b-3859a2f8d677" />


---

## 8. Выводы

### запущены многоконтейнерные приложения, организовано взаимодействие между сервисами, использован Docker Compose для оркестрац,изменена бизнес-логика и инфраструктура проекта; , работа с Redis как с внешним сервисом хранения данных.
---

---

## Приложение. Полный код

### app.py

```python
import time
import redis
import json
import os
from datetime import datetime
from flask import Flask

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
LOG_DIR = os.path.join(BASE_DIR, "logs")
os.makedirs(LOG_DIR, exist_ok=True)
log_path = os.path.join(LOG_DIR, "debug.log")


def log_debug(msg, data=None, hypothesis_id="APP"):
    try:
        with open(log_path, "a", encoding="utf-8") as f:
            f.write(json.dumps({
                "sessionId": "distributed-system",
                "runId": "docker-compose",
                "hypothesisId": hypothesis_id,
                "location": "app.py",
                "message": msg,
                "data": data or {},
                "timestamp": int(time.time() * 1000),
                "datetime": datetime.now().isoformat()
            }, ensure_ascii=False) + "\n")
    except Exception as exc:
        print("Logging error:", exc)


log_debug("Application started")

app = Flask(__name__)
cache = redis.Redis(host="redis", port=6379, decode_responses=True)


def get_hit_count():
    retries = 5
    while True:
        try:
            count = cache.incr("hits")
            log_debug("Counter incremented", {"count": count}, hypothesis_id="COUNTER")
            return count
        except redis.exceptions.ConnectionError as exc:
            log_debug("Redis connection error", {"error": str(exc), "retries_left": retries}, hypothesis_id="REDIS")
            if retries == 0:
                raise exc
            retries -= 1
            time.sleep(0.5)


def get_decision(count):
    if count % 21 == 0:
        return {"title": "Super Prize!", "message": "You are the legendary visitor!", "color": "#8e44ad", "emoji": "👑"}
    if count % 10 == 0:
        return {"title": "Milestone Visitor!", "message": "Congratulations on the round number.", "color": "#2980b9", "emoji": "🎯"}
    if count % 7 == 0:
        return {"title": "Lucky Visitor!", "message": "Luck is on your side today.", "color": "#27ae60", "emoji": "🍀"}
    if count % 2 == 0:
        return {"title": "Even Visitor", "message": "Request processed successfully.", "color": "#555", "emoji": "⚙️"}
    return {"title": "Regular Visitor", "message": "Request processed by distributed system.", "color": "#333", "emoji": "👋"}


@app.route("/")
def hello():
    count = get_hit_count()
    decision = get_decision(count)
    log_debug("Decision selected", {"count": count, "decision": decision["title"]}, hypothesis_id="DECISION")
    return f"""
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Distributed App</title>
        <style>
            body {{ font-family: Arial, sans-serif; background: #f4f6f8; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; }}
            .card {{ background: white; padding: 40px; border-radius: 18px; box-shadow: 0 10px 30px rgba(0,0,0,0.12); text-align: center; max-width: 520px; }}
            .emoji {{ font-size: 64px; }}
            h1 {{ color: {decision["color"]}; }}
            .count {{ font-size: 30px; font-weight: bold; color: {decision["color"]}; }}
            .message {{ font-size: 18px; color: #555; }}
            .footer {{ margin-top: 20px; font-size: 13px; color: #888; }}
        </style>
    </head>
    <body>
        <div class="card">
            <div class="emoji">{decision["emoji"]}</div>
            <h1>{decision["title"]}</h1>
            <p class="message">{decision["message"]}</p>
            <p>Visitors count:</p>
            <div class="count">{count}</div>
            <div class="footer">Logs saved to: <strong>logs/debug.log</strong></div>
        </div>
    </body>
    </html>
    """


@app.route("/health")
def health():
    try:
        cache.ping()
        log_debug("Health check OK", hypothesis_id="HEALTH")
        return {"status": "ok", "redis": "connected"}
    except Exception as exc:
        log_debug("Health check failed", {"error": str(exc)}, hypothesis_id="HEALTH")
        return {"status": "error", "redis": "disconnected"}, 500


if __name__ == "__main__":
    log_debug("Flask application started")
    app.run(host="0.0.0.0", port=5000, debug=True)
```

### Dockerfile

```dockerfile
FROM python:3.9-alpine
ENV TEAM="Sales"
WORKDIR /code
COPY requirements.txt requirements.txt
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

### docker-compose.yml

```yaml
services:
  frontend:
    build: .
    ports:
      - "8000:5000"
    depends_on:
      - redis
    volumes:
      - ./logs:/code/logs
  redis:
    image: redis:alpine
```

### requirements.txt

```txt
Flask==2.0.1
Werkzeug==2.3.7
redis==4.6.0
```
```
