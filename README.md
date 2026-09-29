# Notification Service

Сервис уведомлений платформы Corporation X. Сервис работает как Kafka-консьюмер:
получает события, запрашивает данные пользователя в User Service, формирует
локализованное сообщение и передает его в канал связи, выбранный пользователем.

## Реализованный функционал

- Обработка событий из Kafka-топика `recommendation_received_events`.
- Десериализация события рекомендации:
  `id`, `authorId`, `receiverId`, `createdAt`.
- Получение получателя рекомендации через User Service.
- Повторные запросы к User Service при временных ошибках: до 3 попыток с
  экспоненциальной задержкой.
- Формирование текста уведомления с учетом локали пользователя.
- Выбор реализации `NotificationService` по предпочтительному каналу пользователя.
- Логирование ошибок обработки события и случаев, когда для выбранного канала
  нет доступной реализации.

Сервис не предоставляет REST-эндпоинт для отправки уведомлений: основной поток
обмена данными проходит через Kafka.

## Технологии

- Java 17
- Spring Boot 3
- Spring Kafka
- Spring Cloud OpenFeign
- Gradle
- Redis client
- SMTP и Vonage SDK для интеграций с каналами уведомлений

## Требования

- JDK 17 или Docker
- Доступный Kafka-брокер
- Запущенный User Service
- Redis, если он используется конфигурацией окружения

По умолчанию приложение ожидает Kafka на `localhost:9094`, Redis на
`localhost:6379`, User Service на `localhost:8080` и запускается на порту
`8083`.

## Запуск локально

Соберите приложение из корневой директории проекта:

```shell
.\gradlew.bat clean bootJar
```

Запустите собранный JAR:

```shell
java -jar build\libs\service.jar
```

Для запуска тестов:

```shell
.\gradlew.bat test
```

## Запуск в Docker

Сначала соберите JAR, который используется в `Dockerfile`:

```shell
.\gradlew.bat clean bootJar
```

Соберите Docker-образ:

```shell
docker build -t notification-service .
```

Запустите контейнер, передав адреса зависимостей. Имена хостов должны быть
доступны из Docker-сети:

```shell
docker run --rm --name notification-service `
  -p 8083:8083 `
  -e KAFKA_BOOTSTRAP_SERVERS=host.docker.internal:9094 `
  -e REDIS_HOST=host.docker.internal `
  -e REDIS_PORT=6379 `
  -e USER_SERVICE_HOST=host.docker.internal `
  -e USER_SERVICE_PORT=8080 `
  notification-service
```

Переменные окружения:

| Переменная | Значение по умолчанию | Назначение |
| --- | --- | --- |
| `KAFKA_BOOTSTRAP_SERVERS` | `localhost:9094` | Адрес Kafka |
| `REDIS_HOST` | `localhost` | Хост Redis |
| `REDIS_PORT` | `6379` | Порт Redis |
| `USER_SERVICE_HOST` | `localhost` | Хост User Service |
| `USER_SERVICE_PORT` | `8080` | Порт User Service |
| `PROJECT_SERVICE_HOST` | `localhost` | Хост Project Service |
| `PROJECT_SERVICE_PORT` | `8082` | Порт Project Service |

## Kafka

Сервис слушает топик:

```text
recommendation_received_events
```

Группа потребителей:

```text
notification-service-group
```

Пример события:

```json
{
  "id": 16,
  "authorId": 1,
  "receiverId": 7,
  "createdAt": "2025-11-12T12:06:09"
}
```
