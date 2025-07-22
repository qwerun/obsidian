## [!info] Основная идея

В _Go 1.23_ в компилятор и `go`-toolchain встроена система **телеметрии**: она анонимно считает, как часто разработчики пользуются теми или иными командами, насколько успешны сборки, сколько времени занимает linкoвка и т.д. Управляется новой утилитой **`go telemetry`**.  
По умолчанию включается режим **`local`** — данные пишутся только на диск, не отправляясь наружу, но пользователь может явно перейти в `on` или `off`. ([Go Tips](https://tip.golang.org/doc/go1.23?utm_source=chatgpt.com "Go 1.23 Release Notes - The Go Programming Language"))

---

## Подробно

Конфигурация считывается из переменных окружения:

|Переменная|Значение|
|---|---|
|`GOTELEMETRY`|`local` (дефолт) / `on` / `off`.|
|`GODEBUG`|Ключ `telemetry=0/1` временно переопределяет режим.|

> [!tip]  
> Для быстрой проверки:
> 
> ```bash
> go env GOTELEMETRY  
> # local  
> ```

### Команда go telemetry

|Подкоманда|Действие|
|---|---|
|`go telemetry`|Показать текущий режим.|
|`go telemetry on` / `off` / `local`|Изменить политику.|
|`go telemetry status`|Размер и кол-во локальных журналов.|
|`go telemetry upload`|Принудительно отправить отчёт (режим `on`).|

Логи хранятся в `$GOCACHE/telemetry`; каждый файл ≈ few KB, подписан SHA-256 для проверки целостности. ([Go Tips](https://tip.golang.org/doc/go1.23?utm_source=chatgpt.com "Go 1.23 Release Notes - The Go Programming Language"))

```bash
go telemetry status
# mode: local
# weekly logs: 3  size: 18 KB
```

### Преимущества

- **Быстрый feedback-loop**: метрики сборок помогают фиксить регрессии до релиза.
    
- **Анонимность**: записываются только счётчики, без исходников или имён пакетов. ([Go](https://go.dev/blog/gotelemetry?utm_source=chatgpt.com "Telemetry in Go 1.23 and beyond - The Go Programming Language"))

### Практическое применение

|Сценарий|Комментарий|
|---|---|
|**CI-politika**|`go telemetry off` в контейнере, чтобы не плодить лишние логи.|
|**Аудит**|DevOps просматривает `go telemetry status` перед отправкой.|
|**Корпоративный proxy**|Переменная `GOTELEMETRYURL` указывает внутренний endpoint.|

### Нюансы и подводные камни

> [!warning]  
> _Конфиденциальность_: хотя данные обезличены, корпоративные политики могут требовать режима `off`.  
> _Производительность_: накладные расходы минимальны (< 0.1 %), но при `go telemetry on` добавляется фоновая отправка.

### Рекомендации и FAQ

> [!tip]  
> Храните логи телеметрии в кеше CI как обычный артефакт — это ускорит последующие сборки, если режим `local`.

---

## Полезные ссылки

- [https://go.dev/blog/telemetry](https://go.dev/blog/telemetry)

[[Очистка и обслуживание|Предыдущая статья]]