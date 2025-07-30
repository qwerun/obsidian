> [!info] **Коротко о главном**  
> **OpenAPI** — это машинописная «инструкция» к вашему REST API, понятная людям и программам.  
> **Swagger** — семейство утилит (UI, Editor, Codegen и др.), которые читают эту инструкцию, показывают её красиво, проверяют и даже пишут код.

---

## Краткий конспект

- **OpenAPI** — открытый стандарт описания REST API (актуальная версия 3.1).  
    Файл `.yaml`/`.json` → документация, генерация кода, тесты.
    
- **Swagger** — инструменты, «швейцарский нож» вокруг OpenAPI:  
    визуализация (Swagger UI), онлайн‑редактор, генерация SDK, хостинг (SwaggerHub).
    
- Swagger 2.0 был переименован в OpenAPI 3.0 (2015 г., Linux Foundation).
    
- Swagger 2.0 ↔ OpenAPI 3.x несовместимы, миграция требует конвертации.
    
- Типовые задачи: автодокументация, проверка контрактов, мок‑серверы.
    

---

## 1. Что такое OpenAPI и как им пользуются

**OpenAPI Specification (OAS)** описывает:

|Что описываем|Как выглядит в файле|
|---|---|
|Endpoint|`/users/{id}`|
|Параметры|`path`, `query`, `header`, `cookie`|
|Тела запросов|`requestBody: content: application/json:`|
|Ответы|`responses: '200': content: …`|
|Авторизация|`securitySchemes: bearerAuth:`|

### Практический сценарий «от нуля»

1. **Пишем контракт**  
    Создаём `api.yaml` в Swagger Editor (или VS Code + плагин).
    
    ```yaml
    paths:
      /users/{id}:
        get:
          parameters:
            - name: id
              in: path
              required: true
              schema: { type: integer }
          responses:
            '200':
              description: OK
              content:
                application/json:
                  schema: { $ref: '#/components/schemas/User' }
    ```
    
2. **Соглашение команд**  
    Файл кладём в репозиторий → фронт и бекенд работают «по одной бумажке».
    
3. **Документация**  
    Открываем файл в Swagger UI ➜ получаем интерактивную страницу, где можно  
    «пощёлкать» запросы руками.
    
4. **Генерация кода**  
    `openapi-generator-cli generate -i api.yaml -g typescript-axios -o sdk/`  
    → фронт получает готовый SDK.
    
5. **Тесты‑контракты**  
    В CI запускаем `dredd api.yaml http://localhost:8080` → бекенд не отклоняется от контракта.
    

---

## 2. Что такое Swagger и как им пользуются

|Инструмент|Для чего нужен|Как применить на практике|
|---|---|---|
|**Swagger UI**|HTML‑страница с «песочницей»|`docker run -p 8080:8080 -e SWAGGER_JSON=/api.yaml -v $PWD:/api swaggerapi/swagger-ui`|
|**Swagger Editor**|Онлайн/локальный редактор OAS|`docker run -p 3000:8080 swaggerapi/swagger-editor`|
|**Swagger Codegen**|Старший брат OpenAPI Generator v2|`swagger-codegen generate -i api.yaml -l java -o server/`|
|**SwaggerHub**|SaaS для совместной работы|Храним версии OAS, обсуждаем изменения, получаем mock‑URL|

> [!tip]  
> Думайте о Swagger как о Postman, заточенном под один формат (OpenAPI).

---

## 3. Ключевые отличия OpenAPI 3.1 vs Swagger 2.0

|Критерий|Swagger 2.0|OpenAPI 3.1|
|---|---|---|
|Совместимость с JSON Schema|частичная|**полная**|
|Media Types|`application/json` по умолч.|любой (`text/plain`, `image/png`, …)|
|Переиспользование компонентов|`definitions/parameters`|`components` (схемы, ответы, хэдеры, …)|
|Поле `servers` (базовые URL)|—|✅|
|Расширяемость|`x-…`|`x-…` + аннотации JSON Schema|

---

## 4. Типовые сценарии применения

- **Автодокументация**: Swagger UI, ReDoc.
    
- **SDK‑фабрика**: OpenAPI Generator → клиенты на TS, Java, Go, Python, …
    
- **Мок‑сервер**: `prism mock -d api.yaml` — фронт начинает работу без бекенда.
    
- **Контрактное тестирование**: Dredd, Pact, Optic.
    
- **CI‑защита**: линтеры (`spectral lint api.yaml`) ловят breaking changes до merge.
    

---

## 5. Подводные камни

> [!warning]  
> На что чаще всего жалуются команды:

- **Миграция 2.0 → 3.x**: нужно переписать `parameters`, убрать `definitions`.
    
- **Несинхронные тулзы**: не всё ПО уже читает 3.1 (пример: старые версии AWS API Gateway).
    
- **Огромные спецификации**: дробите на модули и собирайте `swagger-cli bundle`.
    
- **Разное понимание nullable/oneOf**: проверяйте рендер в Swagger UI и ReDoc.
    

---

## Мини‑пример: “Hello API” в 3.1

```yaml
openapi: 3.1.0
info: { title: Hello API, version: 1.0.0 }
servers: [{ url: https://api.example.com/v1 }]
paths:
  /hello:
    get:
      summary: Say hi
      responses:
        '200':
          description: Plain greeting
          content:
            text/plain:
              schema: { type: string, example: "Hi!" }
```

Сохраните как `hello.yaml` и откройте в Swagger UI — сразу получите кнопочку **Try it out**.

---

> [!faq]  
> **OpenAPI и Swagger — это одно и то же?**  
> Нет. OpenAPI — стандарт; Swagger — инструменты, которые с ним работают.
> 
> **Можно ли Swagger UI + OpenAPI 3.1?**  
> Да, Swagger UI v4+ полностью поддерживает 3.1.
> 
> **Зачем всё это, если у меня Postman?**  
> Postman читает OpenAPI, но не генерирует сервер/SDK. OAS = «единственный источник истины» для всех фаз разработки.

---

## Полезные ссылки

- [Официальная спецификация OpenAPI 3.1](https://spec.openapis.org/oas/v3.1.1.html)
    
- [Swagger UI GitHub](https://github.com/swagger-api/swagger-ui)
    
- [OpenAPI Generator](https://openapi-generator.tech/)
    
- [Lint‑/Mock‑/Test‑утилиты список](https://openapi.tools/)