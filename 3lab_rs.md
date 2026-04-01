# Лабораторная работа 3.1
## Организация асинхронного взаимодействия микросервисов с помощью брокера сообщений

* **Студент:** *Губанова Светлана Алексеевна*
* **Группа:** *ЦИБ-241*

---

## Постановка задачи

Разработать систему микросервисов для обработки пользовательских запросов с использованием двух подходов к взаимодействию:

1. **Синхронное взаимодейшение** — прямое общение между сервисами через gRPC в режиме "запрос-ответ".
2. **Асинхронное взаимодействие** — общение через брокер сообщений RabbitMQ, где Producer отправляет сообщения в очередь, а Consumer забирает их для последующей обработки.

---

## Вариант 1: Расчет скидки

**Номер варианта:** 1

**Задания для реализации:**

1. **Расчет скидки**  
   Producer отправляет цену товара.  
   gRPC сервис применяет скидку 15% и возвращает новую цену.

2. **Генерация аватара**  
   Producer отправляет email.  
   gRPC сервис генерирует для него ссылку на аватар Gravatar.

3. **Перевод в Markdown**  
   Producer отправляет текст.  
   gRPC сервис оборачивает его в базовые Markdown теги (делает жирным) и возвращает результат.

---
## Архитектура

### Часть 1. Синхронное взаимодействие (gRPC)

#### Диаграмма синхронного взаимодействия

```mermaid
sequenceDiagram
    participant Client as gRPC Client
    participant Server as gRPC Server
    
    Note over Client,Server: Установка соединения
    
    Client->>Server: ProcessMessage(цена: 1000 руб.)
    activate Server
    
    Note over Server: Обработка запроса<br/>Расчет скидки 15%
    
    Server-->>Client: MessageResponse(850 руб.)
    deactivate Server
    
    rect rgb(200, 220, 240)
        Note over Client,Server: СИНХРОННЫЙ РЕЖИМ<br/>Клиент ожидает ответа<br/>Сервер занят на время обработки
    end
```

#### Описание синхронного взаимодействия

**Архитектура:**
- **gRPC Client** — инициатор запроса, отправляет цену товара
- **gRPC Server** — обработчик запроса, применяет скидку 15%

**Логика работы:**
1. Клиент устанавливает соединение с сервером
2. Клиент отправляет запрос с ценой товара
3. Сервер получает запрос и блокируется на время обработки
4. Сервер рассчитывает цену со скидкой 15%
5. Сервер возвращает результат клиенту
6. Клиент получает ответ, соединение закрывается

---

### Часть 2. Асинхронное взаимодействие (RabbitMQ + CloudAMQP)

#### Диаграмма асинхронного взаимодействия

```mermaid
sequenceDiagram
    participant Producer as Producer<br/>(Отправитель)
    participant RabbitMQ as CloudAMQP<br/>(Брокер сообщений)
    participant Consumer as Consumer<br/>(Получатель)
    participant GRPC as gRPC Server<br/>(Сервис обработки)
    
    Note over Producer,RabbitMQ: Отправка сообщения
    
    Producer->>RabbitMQ: Отправить запрос
    RabbitMQ-->>Producer: Подтверждение
    Producer-->>Producer: Producer завершает работу
    
    Note over RabbitMQ: Хранение сообщения<br/>в очереди task_queue
    
    Note over Consumer,RabbitMQ: Получение сообщения
    
    Consumer->>RabbitMQ: Забрать сообщение
    RabbitMQ-->>Consumer: Сообщение
    
    Note over Consumer,GRPC: Обработка через gRPC
    
    Consumer->>GRPC: ProcessMessage(запрос)
    GRPC-->>Consumer: Ответ
    
    Consumer->>RabbitMQ: Подтверждение (ack)
    
    rect rgb(200, 240, 200)
        Note over Producer,Consumer: АСИНХРОННЫЙ РЕЖИМ<br/>Producer не ждет ответа<br/>Consumer обрабатывает независимо
    end
```
---

## Преимущества асинхронного подхода для варианта "Расчет скидки"

| Характеристика | Синхронный (gRPC) | Асинхронный (RabbitMQ + CloudAMQP) |
|----------------|-------------------|-------------------------------------|
| **Ожидание ответа** | Клиент ждет ответа | Producer не ждет ответа |
| **Отказ сервера** | Клиент получает ошибку | Сообщение сохраняется в очереди |
| **Масштабирование** | Нужно увеличивать число серверов | Можно добавить несколько Consumer |
| **Нагрузка** | Сервер обрабатывает запросы синхронно | Сообщения накапливаются в очереди |
| **Перезапуск** | Потеря запросов | Сообщения сохраняются (durable) |
| **Развертывание** | Требуется локальный сервер | Облачный брокер, доступен из любой точки |

---

## Ход выполнения работы

### Этап 1. Создание виртуального окружения и установка зависимостей

**Выполненные команды:**

```bash
# Создание папки проекта
mkdir lab3_microservices
cd lab3_microservices

# Создание виртуального окружения
python -m venv venv

# Активация виртуального окружения (Windows)
venv\Scripts\activate

# Установка необходимых библиотек
pip install grpcio grpcio-tools pika
```

**Результат:**  
Создана структура проекта и установлены все зависимости.

---

### Этап 2. Создание файла контракта .proto

**Создан файл:** `message_service.proto`

**Код:**

```protobuf
syntax = "proto3";

package message;

// Определение сервиса
service MessageService {
    // Метод, который принимает запрос и возвращает ответ
    rpc ProcessMessage (MessageRequest) returns (MessageResponse) {}
}

// Структура сообщения-запроса
message MessageRequest {
    string text = 1;
}

// Структура сообщения-ответа
message MessageResponse {
    string processed_text = 1;
}
```

**Назначение:**  
Файл описывает контракт gRPC сервиса. Сервис `MessageService` содержит один метод `ProcessMessage`, который принимает строку текста и возвращает обработанную строку.

---

### Этап 3. Генерация gRPC кода

**Выполненная команда:**

```bash
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. message_service.proto
```

**Результат:**  
Сгенерированы файлы:
- `message_service_pb2.py` — содержит классы для сообщений
- `message_service_pb2_grpc.py` — содержит классы для сервиса

---

### Этап 4. Создание gRPC сервера

**Создан файл:** `grpc_server.py`

**Код:**

```python
import grpc
from concurrent import futures
import hashlib
import message_service_pb2
import message_service_pb2_grpc

class MessageService(message_service_pb2_grpc.MessageServiceServicer):
    
    def ProcessMessage(self, request, context):
        """
        Универсальный метод, обрабатывающий все три варианта заданий:
        - Вариант 1: Расчет скидки 15% (если прислали число)
        - Вариант 2: Генерация аватара Gravatar (если прислали email)
        - Вариант 3: Перевод в Markdown (если прислали текст)
        """
        text = request.text
        print(f"\n[СЕРВЕР] Получено: {text}")
        
        # ========== ВАРИАНТ 1: Расчет скидки ==========
        # Проверяем, является ли текст числом
        try:
            price = float(text)
            discount_percent = 15
            discounted_price = price * (1 - discount_percent / 100)
            result = f"【Вариант 1: Расчет скидки】 Исходная цена: {price} руб. | Скидка {discount_percent}% | Цена со скидкой: {discounted_price:.2f} руб."
            print(f"[СЕРВЕР] Вариант 1: {price} руб. -> {discounted_price:.2f} руб.")
            return message_service_pb2.MessageResponse(processed_text=result)
        except ValueError:
            pass
        
        # ========== ВАРИАНТ 2: Генерация аватара ==========
        # Проверяем, похоже ли на email (содержит @ и точку)
        if '@' in text and '.' in text:
            email = text.strip().lower()
            # Создаем MD5 хеш от email (как требует Gravatar)
            email_hash = hashlib.md5(email.encode()).hexdigest()
            gravatar_url = f"https://www.gravatar.com/avatar/{email_hash}?d=identicon&s=200"
            result = f"【Вариант 2: Генерация аватара】 Email: {email}\nGravatar URL: {gravatar_url}"
            print(f"[СЕРВЕР] Вариант 2: {email} -> {gravatar_url}")
            return message_service_pb2.MessageResponse(processed_text=result)
        
        # ========== ВАРИАНТ 3: Перевод в Markdown ==========
        # Всё остальное обрабатываем как текст
        markdown_text = f"**{text}**"  # делаем текст жирным
        result = f"【Вариант 3: Перевод в Markdown】 Исходный текст: \"{text}\"\nMarkdown результат: {markdown_text}"
        print(f"[СЕРВЕР] Вариант 3: \"{text}\" -> **{text}**")
        return message_service_pb2.MessageResponse(processed_text=result)

def serve():
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=10))
    message_service_pb2_grpc.add_MessageServiceServicer_to_server(MessageService(), server)
    server.add_insecure_port('[::]:50051')
    print("=" * 70)
    print("gRPC СЕРВЕР ЗАПУЩЕН (порт 50051)")
    print("-" * 70)
    print("Доступные варианты заданий:")
    print("  1. РАСЧЕТ СКИДКИ 15%   -> отправьте число (например: 1000)")
    print("  2. ГЕНЕРАЦИЯ АВАТАРА   -> отправьте email (например: test@mail.ru)")
    print("  3. ПЕРЕВОД В MARKDOWN  -> отправьте текст (например: Привет мир)")
    print("=" * 70)
    server.start()
    server.wait_for_termination()

if __name__ == '__main__':
    serve()
```

**Логика работы сервера:**
1. Сервер запускается на порту 50051 и ждет запросы
2. При получении сообщения определяет его тип:
   - Если сообщение можно преобразовать в число → применяет скидку 15%
   - Если сообщение содержит @ и точку → генерирует ссылку на Gravatar
   - В остальных случаях → оборачивает текст в жирное начертание Markdown
3. Возвращает обработанный результат

---

### Этап 5. Создание gRPC клиента для тестирования

**Создан файл:** `grpc_client.py`

**Код:**

```python
import grpc
import message_service_pb2
import message_service_pb2_grpc

def run():
    with grpc.insecure_channel('localhost:50051') as channel:
        stub = message_service_pb2_grpc.MessageServiceStub(channel)
        
        # Тестируем с ценой 1000 рублей
        price = "1000"
        response = stub.ProcessMessage(message_service_pb2.MessageRequest(text=price))
        
        print("=" * 50)
        print("Тестирование gRPC сервера")
        print(f"Отправлена цена: {price} руб.")
        print(f"Ответ сервера: {response.processed_text}")
        print("=" * 50)

if __name__ == '__main__':
    run()
```

**Назначение:**  
Клиент используется для проверки работоспособности gRPC сервера напрямую, без RabbitMQ.

---

### Этап 6. Настройка облачного брокера CloudAMQP

**Выполненные действия:**

1. Зарегистрировался на сайте https://www.cloudamqp.com/
2. Создал бесплатный инстанс RabbitMQ
3. Получил параметры подключения:

| Параметр | Значение |
|----------|---------|
| **AMQP URL** | `amqps://jnxozfti:sifdYr4KwdMiYyOF1vCwzC0udiNQxlEc@raccoon.lmq.cloudamqp.com/jnxozfti` |
| **Сервер** | `raccoon.lmq.cloudamqp.com` |
| **Порт** | `5672` |
| **Пользователь** | `jnxozfti` |

**Преимущества CloudAMQP:**
- Не требует локальной установки Docker или RabbitMQ
- Предоставляет веб-интерфейс для мониторинга
- Сообщения сохраняются в облаке

---

### Этап 7. Создание Producer (отправитель сообщений)

**Создан файл:** `producer.py`

**Код:**

```python
import pika
import sys

# AMQP URL из CloudAMQP
AMQP_URL = "amqps://jnxozfti:sifdYr4KwdMiYyOF1vCwzC0udiNQxlEc@raccoon.lmq.cloudamqp.com/jnxozfti"

def main():
    # Подключаемся к CloudAMQP
    params = pika.URLParameters(AMQP_URL)
    connection = pika.BlockingConnection(params)
    channel = connection.channel()
    
    # Создаем очередь (durable=True - сообщения сохраняются)
    channel.queue_declare(queue='task_queue', durable=True)
    
    # Получаем сообщение из аргументов командной строки
    if len(sys.argv) > 1:
        message = sys.argv[1]
    else:
        message = "1000"
    
    # Отправляем сообщение в очередь
    channel.basic_publish(
        exchange='',
        routing_key='task_queue',
        body=message,
        properties=pika.BasicProperties(
            delivery_mode=2,  # делаем сообщение постоянным
        )
    )
    
    print(f"[x] Отправлено сообщение: '{message}'")
    connection.close()

if __name__ == '__main__':
    main()
```

**Логика работы Producer:**
1. Подключается к CloudAMQP по полученному URL
2. Создает очередь `task_queue` (если её нет)
3. Отправляет сообщение (цену, email или текст)
4. Закрывает соединение

---

### Этап 8. Создание Consumer (получатель и обработчик)

**Создан файл:** `consumer.py`

**Код:**

```python
import pika
import grpc
import message_service_pb2
import message_service_pb2_grpc

# AMQP URL из CloudAMQP (тот же, что и в producer.py)
AMQP_URL = "amqps://jnxozfti:sifdYr4KwdMiYyOF1vCwzC0udiNQxlEc@raccoon.lmq.cloudamqp.com/jnxozfti"

def process_message_via_grpc(text):
    """Вызов gRPC сервиса для обработки сообщения"""
    try:
        with grpc.insecure_channel('localhost:50051') as channel:
            stub = message_service_pb2_grpc.MessageServiceStub(channel)
            response = stub.ProcessMessage(message_service_pb2.MessageRequest(text=text))
            return response.processed_text
    except grpc.RpcError as e:
        return f"Ошибка вызова gRPC: {e.details()}"

def main():
    # Подключаемся к CloudAMQP
    params = pika.URLParameters(AMQP_URL)
    connection = pika.BlockingConnection(params)
    channel = connection.channel()
    
    # Создаем очередь (должна существовать)
    channel.queue_declare(queue='task_queue', durable=True)
    
    print('=' * 50)
    print('Consumer запущен. Ожидание сообщений...')
    print('Для выхода нажмите CTRL+C')
    print('=' * 50)
    
    def callback(ch, method, properties, body):
        message_text = body.decode()
        print(f"\n[x] Получено сообщение: {message_text}")
        
        # Обрабатываем сообщение через gRPC
        processed_result = process_message_via_grpc(message_text)
        print(f"[✓] Результат: {processed_result}")
        
        # Подтверждаем обработку сообщения
        ch.basic_ack(delivery_tag=method.delivery_tag)
    
    # Настройка обработки сообщений
    channel.basic_qos(prefetch_count=1)  # Обрабатываем по одному сообщению
    channel.basic_consume(queue='task_queue', on_message_callback=callback)
    
    channel.start_consuming()

if __name__ == '__main__':
    main()
```

**Логика работы Consumer:**
1. Подключается к CloudAMQP
2. Начинает слушать очередь `task_queue`
3. При получении сообщения вызывает gRPC сервис
4. Получает результат и выводит его
5. Подтверждает обработку сообщения (удаляет из очереди)

---

### Этап 9. Запуск и тестирование системы

**Запуск в трех терминалах:**

**Терминал 1 - gRPC сервер:**
```bash
cd C:\Users\LENOVO\Downloads\lab3_microservices
venv\Scripts\activate
python grpc_server.py
```
<img width="691" height="427" alt="image" src="https://github.com/user-attachments/assets/02cbee40-506c-454d-a5ef-10253e5657f2" />

**Терминал 2 - Consumer:**
```bash
cd C:\Users\LENOVO\Downloads\lab3_microservices
venv\Scripts\activate
python consumer.py
```
<img width="689" height="183" alt="image" src="https://github.com/user-attachments/assets/0928256c-6113-4479-8735-a5b9fff18cb5" />

**Терминал 3 - Producer (отправка сообщений):**
```bash
cd C:\Users\LENOVO\Downloads\lab3_microservices
venv\Scripts\activate
python producer.py "1500"
```
<img width="671" height="202" alt="image" src="https://github.com/user-attachments/assets/f678a73b-9217-4e82-b464-9342f65d9f68" />

---


## Стек технологий

| Компонент | Технология | Назначение |
|-----------|------------|------------|
| **Язык программирования** | Python 3.10+ | Реализация всех сервисов |
| **Синхронное взаимодействие** | gRPC (grpcio, grpcio-tools) | Прямое взаимодействие микросервисов |
| **Асинхронное взаимодействие** | RabbitMQ (pika) | Брокер сообщений |
| **Облачный хостинг RabbitMQ** | CloudAMQP | Бесплатный облачный брокер |
| **Протокол** | Protocol Buffers | Определение контракта сервиса |

**Установка зависимостей:**
```bash
pip install grpcio grpcio-tools pika
```

---

## Результаты тестирования

### Проверка Варианта 1 (Расчет скидки)

**Отправка:**
```bash
python producer.py "1000"
```

**Результат в Consumer:**

<img width="664" height="90" alt="image" src="https://github.com/user-attachments/assets/251411ef-1231-4618-a88c-88dd292f0a4f" />


---

### Проверка Варианта 2 (Генерация аватара)

**Отправка:**
```bash
python producer.py "ivan@example.com"
```

**Результат в Consumer:**

<img width="678" height="118" alt="image" src="https://github.com/user-attachments/assets/9bd86c8b-a6d1-4452-9682-3e55015a5654" />

---

### Проверка Варианта 3 (Перевод в Markdown)

**Отправка:**
```bash
python producer.py "Привет, мир"
```

**Результат в Consumer:**
<img width="683" height="126" alt="image" src="https://github.com/user-attachments/assets/c93d21af-9185-4a28-a932-8081709f2ea4" />


---

## Заключение
Изучены и реализованы два ключевых подхода к взаимодействию между сервисами: синхронное прямое взаимодействие с использованием gRPC и асинхронное взаимодействие через брокера сообщений RabbitMQ. Освоить развертывание инфраструктурных компонентов с помощью Docker.
