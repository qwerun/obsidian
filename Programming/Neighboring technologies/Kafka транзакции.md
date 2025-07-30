> [!info] Основная идея  
> Транзакции в Kafka позволяют обеспечить семантику [[Exactly-Once]], критичную для надёжной обработки сообщений в распределённых системах. Для микросервисов это особенно важно при работе с базами данных, коммуникацией через топики и реализацией паттернов Saga и Transactional Outbox.

---

## Краткий конспект

- Продюсеры могут отправлять несколько сообщений в разные топики как одну атомарную операцию.
    
- Transaction Coordinator отслеживает статус транзакций.
    
- В микросервисах транзакции применяются в паттернах Saga и Transactional Outbox.
    
- Требует конфигурации: `EnableIdempotence=true`, `acks=all`, `TransactionalID`.
    
- Необходимо использовать `read_committed` на стороне потребителя.
    
- Проблемы: сложность отладки, таймауты, rebalancing, увеличение задержек.
    

---

## Подробно

### Принцип работы транзакций в Apache Kafka

Kafka предоставляет модель **атомарной доставки сообщений** с гарантией, что они будут либо доставлены все, либо ни одного. Это достигается через:

1. **[[Идемпотентность]]** — включается флагом `EnableIdempotence=true`, чтобы повторно отправленные сообщения не дублировались.
    
2. **Transactional Coordinator** — компонент на брокере, отслеживающий статус транзакций.
    
3. **Fencing** — механизм, исключающий конкуренцию продюсеров с одинаковым `TransactionalID`.
    

#### Этапы транзакции:

1. `BeginTransaction()` — начинает новую транзакцию.
    
2. `SendMessage()` — отправляет одно или несколько сообщений.
    
3. `CommitTransaction()` — фиксирует транзакцию.
    
4. `AbortTransaction()` — откатывает транзакцию.
    

> [!tip]  
> Даже одна запись в один топик может быть обёрнута в транзакцию, если критично соблюсти семантику [[Exactly-Once]].

---

### [[Exactly-Once]] Semantics (EOS)

#### Варианты доставки:

- **At Most Once** — сообщение может потеряться.
    
- **At Least Once** — возможны дубликаты.
    
- **Exactly Once** — сообщение доставляется ровно один раз.
    

Kafka реализует EOS путём сочетания:

- [[Идемпотентность|идемпотентных]] продюсеров (`EnableIdempotence`);
    
- транзакций;
    
- `read_committed` на стороне consumer;
    
- fencing идентификаторов продюсеров.
    

> [!example]  
> Kafka [[exactly-once]] semantics описывает, как это работает в деталях.

---

### Пример конфигурации transactional-продюсера в Go (sarama)

```go
config := sarama.NewConfig()
config.Producer.RequiredAcks = sarama.WaitForAll
config.Producer.Idempotent = true
config.Producer.Return.Successes = true
config.Producer.Transaction.ID = "orders-service-tx"
```

> [!warning]  
> Если не установить `Transaction.ID`, транзакции не будут работать.

---

### Пример транзакции: Begin → Send → Commit

```go
producer, err := sarama.NewSyncProducer([]string{"localhost:9092"}, config)
txnProducer := producer.(sarama.TransactionalProducer)

err = txnProducer.BeginTxn()
if err != nil { log.Fatal(err) }

msg := &sarama.ProducerMessage{
    Topic: "orders",
    Key:   sarama.StringEncoder("order123"),
    Value: sarama.StringEncoder("confirmed"),
}

if err := txnProducer.SendMessage(msg); err != nil {
    _ = txnProducer.AbortTxn()
    log.Fatal(err)
}

if err := txnProducer.CommitTxn(); err != nil {
    _ = txnProducer.AbortTxn()
    log.Fatal(err)
}
```

---

### Пример consumer с `read_committed`

```go
config := sarama.NewConfig()
config.Consumer.Group.Rebalance.Strategy = sarama.BalanceStrategyRoundRobin
config.Consumer.IsolationLevel = sarama.ReadCommitted

consumerGroup, _ := sarama.NewConsumerGroup([]string{"localhost:9092"}, "billing", config)
```

> [!tip]  
> Без `ReadCommitted` потребитель увидит даже несогласованные сообщения.

---

### Транзакции в микросервисах

#### Паттерн Saga

- Каждая операция — отдельная локальная транзакция.
    
- Компенсация ошибок делается через обратные действия.
    
- Kafka используется как оркестратор или хореограф событий.
    

#### Transactional Outbox

**Transactional Outbox** — это паттерн, позволяющий гарантировать согласованную доставку событий в Kafka при выполнении локальной транзакции в микросервисе. Он решает классическую проблему **dual write**: когда нужно и сохранить данные в базу, и отправить сообщение в брокер сообщений, но нельзя сделать это атомарно в одной распределённой транзакции.

---

##### Как работает паттерн:

1. **Бизнес-операция** (например, создание заказа) записывается в основную таблицу (например, `orders`).
2. **Событие-домино** (например, `OrderCreated`) записывается в таблицу `outbox` **в рамках той же транзакции** БД.
3. После коммита транзакции, **фоновый обработчик** (poller) читает записи из `outbox`, отправляет их в Kafka и помечает как отправленные (или удаляет).

```text
BEGIN;

INSERT INTO orders (id, status) VALUES ('123', 'created');
INSERT INTO outbox (aggregate_id, event_type, payload) 
VALUES ('123', 'OrderCreated', '{...}');

COMMIT;
```

---

##### Преимущества:

- ✅ Нет необходимости использовать сложные распределённые транзакции (2PC).
- ✅ Гарантированная доставка: сообщение публикуется только после успешного коммита.
- ✅ Простота отката: при падении транзакции `outbox` не будет содержать мусора.
- ✅ Хорошо работает с [[Exactly-Once ]]транзакциями Kafka.

---

##### Реализация в Go (упрощённо):

```go
// 1. В рамках бизнес-транзакции
tx := db.Begin()

tx.Exec("INSERT INTO orders ...")
tx.Exec("INSERT INTO outbox (event_type, payload) VALUES (?, ?)", "OrderCreated", eventJSON)

tx.Commit()

// 2. Фоновый publisher
rows := db.Query("SELECT * FROM outbox WHERE published = false")
for rows.Next() {
    // читаем payload, отправляем в Kafka
    producer.SendMessage(...)
    db.Exec("UPDATE outbox SET published = true WHERE id = ?", row.ID)
}
```

> [!tip]
> Используй Kafka-транзакции для отправки сообщений из outbox, если нужно [[Exactly-Once]] поведение на этапе публикации.

---

##### Возможные сложности:

- Нужно следить за ростом таблицы `outbox` — потребуется ретеншн.
- Обработка может быть не мгновенной → важно учитывать задержку между коммитом БД и публикацией.
- Требует надёжного retry-механизма и idempotent-семантики при повторной публикации.

> [!example]
> [[Transactional Outbox]] часто сочетается с Kafka [[exactly-once]] semantics для построения надёжных интеграций между сервисами без использования 2PC.


---

### Преимущества и ограничения

#### Преимущества:

- Гарантия целостности (все сообщения доставлены или ни одного).
    
- Исключение дубликатов.
    
- Позволяет строить согласованные цепочки микросервисов.
    

#### Ограничения:

1. **Latency** — транзакции добавляют накладные расходы.
    
2. **Rebalancing** — во время балансировки могут быть проблемы с повторной инициализацией продюсера.
    
3. **Timeouts** — необходимо учитывать `transaction.timeout.ms`.
    
4. **Fencing** — старые продюсеры с тем же ID будут заблокированы.
    
5. **Сложность отладки** — требует мониторинга и логирования статусов транзакций.
    

---

### Мониторинг и отладка

#### Метрики продюсера:

- `kafka.producer.transaction.abort-total`
    
- `kafka.producer.transaction.commit-total`
    
- `kafka.producer.record-send-total`
    

#### Метрики консьюмера:

- `kafka.consumer.records-lag`
    
- `kafka.consumer.commit-latency-avg`
    

#### Best practices:

- Всегда логируйте ошибки `CommitTxn`, `AbortTxn`.
    
- Используйте retry с backoff для `SendMessage`.
    
- Отслеживайте задержки транзакций (метрика commit latency).
    
- Настройте алерты на `transaction.timeout.ms`.
    

> [!tip]  
> Используйте отдельный `TransactionalID` на каждый сервис или инстанс, чтобы избежать fencing при рестартах.

---

> [!faq]  
> **Q:** Что произойдёт, если продюсер умрёт до CommitTxn?  
> **A:** Сообщения не будут доставлены — они останутся в pending и будут отброшены брокером.
> 
> **Q:** Можно ли посылать в разные топики в одной транзакции?  
> **A:** Да, Kafka поддерживает multi-topic и multi-partition транзакции.
> 
> **Q:** Что если consumer не использует `read_committed`?  
> **A:** Он может увидеть несогласованные сообщения, которые потом будут откатаны.
> 
> **Q:** Нужно ли использовать EOS всегда?  
> **A:** Нет. Только если критична семантика [[Exactly-Once]]. В других случаях достаточно [[Идемпотентность|идемпотентности]].

---

## Полезные ссылки

- [https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/](https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/)
    
- [https://microservices.io/patterns/data/saga.html](https://microservices.io/patterns/data/saga.html)
    
- [https://developer.confluent.io/courses/microservices/the-transactional-outbox-pattern/](https://developer.confluent.io/courses/microservices/the-transactional-outbox-pattern/)
    
- [https://www.andreamedda.com/posts/play-with-sarama-transactional-api/](https://www.andreamedda.com/posts/play-with-sarama-transactional-api/)
    
- [https://kafka.apache.org/documentation/#transactions](https://kafka.apache.org/documentation/#transactions)
    
- [https://github.com/Shopify/sarama](https://github.com/Shopify/sarama)