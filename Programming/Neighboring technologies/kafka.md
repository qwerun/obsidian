> [!info]  
> Apache Kafka помогает компаниям надёжно обрабатывать большие объёмы событий в реальном времени. 

---

## Краткий конспект

- Kafka — система для обмена сообщениями и событийной передачи данных между сервисами.
- Ключевые сущности: **Producer**, **Consumer**, **Topic**, **Partition**, **Consumer Group**.
- Поддерживает горизонтальное масштабирование и обработку в реальном времени.
- Используется в high-load системах для логирования, аналитики, стриминга.
- Обеспечивает отказоустойчивость через репликацию и устойчивое хранение.
- Простой API и клиенты под Go, Java, Python и другие языки.
- Часто заменяет очереди RabbitMQ, Redis Streams в event-driven архитектурах.

---

## Что такое Apache Kafka?

Apache Kafka — распределённая платформа потоковой передачи данных, изначально разработанная в LinkedIn и переданная в Apache Software Foundation. Kafka обеспечивает надёжную доставку сообщений между компонентами систем, которые могут быть написаны на разных языках и находиться в разных зонах.

### Когда выбирают Kafka?

- Когда нужно **обрабатывать миллионы сообщений в секунду**.
- Если данные должны быть доступны **в реальном времени** и **надежно сохраняться**.
- В случае **микросервисной архитектуры** для организации слабосвязанных компонентов.

### Примеры использования:

1. **Логирование событий пользователей** в e-commerce: действия пользователя пишутся в Kafka, дальше — в clickstream-аналитику.
2. **Обработка транзакций** в финтехе: Kafka обеспечивает надёжную доставку событий между платёжными сервисами.
3. **Мониторинг и алертинг**: системы метрик отправляют данные в Kafka, а из неё читают дашборды и алерты.

---

## Основные компоненты

### 🟢 Producer

**Producer** — это приложение, которое отправляет сообщения в Kafka.

```go
writer := kafka.NewWriter(kafka.WriterConfig{
	Brokers: []string{"localhost:9092"},
	Topic:   "orders",
})
err := writer.WriteMessages(ctx, kafka.Message{Value: []byte("order123")})
```

> [[FAQ!]] *Вопрос на собеседовании:*  
> *Как producer определяет, в какую партицию отправить сообщение?*  
> ✅ *Ответ:* Через key-партиционирование (по умолчанию по hash от ключа), либо round-robin.

---

### 🔵 Consumer

**Consumer** — это приложение, которое читает сообщения из Kafka.

```go
reader := kafka.NewReader(kafka.ReaderConfig{
	Brokers: []string{"localhost:9092"},
	Topic:   "orders",
	GroupID: "billing-service",
})
msg, _ := reader.ReadMessage(ctx)
fmt.Println(string(msg.Value))
```

> ❓ *Вопрос на собеседовании:*  
> *Как consumer понимает, какие сообщения он уже обработал?*  
> ✅ *Ответ:* Kafka отслеживает offset; consumer может его коммитить вручную или автоматически.

---

### 📦 Topic

**Topic** — логическая категория сообщений (аналог "темы").

```bash
kafka-topics.sh --create --topic payments --bootstrap-server localhost:9092
```

> ❓ *Вопрос на собеседовании:*  
> *Можно ли изменить количество партиций в topic?*  
> ✅ *Ответ:* Да, но только увеличить (`--alter --partitions N`), уменьшить нельзя.

---

### 📊 Partition

**Partition** — физическое распределение сообщений внутри topic.

- Позволяет обрабатывать сообщения параллельно.
- Каждая партиция — упорядоченный журнал.

```bash
kafka-topics.sh --describe --topic payments --bootstrap-server localhost:9092
```

> ❓ *Вопрос на собеседовании:*  
> *Что такое offset?*  
> ✅ *Ответ:* Порядковый номер сообщения в партиции, служит для позиционирования чтения.

---

### 👥 Consumer Group

**Consumer Group** — группа consumer'ов, совместно читающих сообщения из одного topic.

- Каждое сообщение доставляется только одному consumer'у внутри группы.
- Обеспечивает масштабирование по числу партиций.

```bash
kafka-consumer-groups.sh --describe --group billing-service --bootstrap-server localhost:9092
```

> ❓ *Вопрос на собеседовании:*  
> *Что произойдёт при увеличении числа consumer'ов в группе?*  
> ✅ *Ответ:* Партиции перераспределяются между ними; если consumer'ов больше, чем партиций — часть будет простаивать.

---

## Почему Kafka популярна

- **Горизонтальное масштабирование** — можно добавить брокеры и партиции без даунтайма.
- **Надёжность** — поддержка репликации и сохранение на диск.
- **Обработка в реальном времени** — минимальная задержка.
- **Совместимость с разными языками и фреймворками** — клиенты под Go, Java, Python и другие.
- **Поддержка реплейных сценариев** — можно перечитывать данные из любой позиции.

---

## Подводные камни

> [!warning]  
> - **Высокая задержка на чтение** при большом количестве consumer'ов с разными offset.  
> - **Сложность кросс-регионной репликации** и настройки geo-disaster tolerance.  
> - **Зависимость от дискового I/O** — нужны быстрые диски.  
> - **Порядок сообщений сохраняется только внутри одной партиции.**

---

> [!faq]  
> **Q:** Kafka — это очередь или лог?  
> **A:** Это распределённый лог (commit log) с возможностью "очереди" через consumer groups.  
>  
> **Q:** Что если consumer упал?  
> **A:** После рестарта он продолжит с сохранённого offset (если коммит происходил).  
>  
> **Q:** Можно ли читать одновременно из нескольких topic?  
> **A:** Да, Kafka client позволяет подписаться на несколько тем.  
>  
> **Q:** Какой язык предпочтителен для работы с Kafka?  
> **A:** Kafka нативно написана на Java, но для Go есть mature-клиенты (`segmentio/kafka-go`, `confluent-kafka-go`).  
>  
> **Q:** Что будет, если producer не укажет ключ?  
> **A:** Kafka по умолчанию применит round-robin распределение по партициям.

---

## Полезные ссылки

- [Apache Kafka Documentation](https://kafka.apache.org/documentation/?utm_source=chatgpt.com)  
- [Kafka Benefits and Use Cases – Confluent](https://www.confluent.io/learn/apache-kafka-benefits-and-use-cases/?utm_source=chatgpt.com)  
- [Kafka Consumer Design – Confluent Docs](https://docs.confluent.io/kafka/design/consumer-design.html?utm_source=chatgpt.com)  
- [Kafka Interview Q&A – Simplilearn](https://www.simplilearn.com/kafka-interview-questions-and-answers-article?utm_source=chatgpt.com)  
- [Kafka Interview Questions – GeeksforGeeks](https://www.geeksforgeeks.org/kafka-interview-questions/?utm_source=chatgpt.com)  
