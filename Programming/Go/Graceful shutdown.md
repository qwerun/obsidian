> [!info] **Основная идея**  
> **Graceful shutdown** — контролируемое завершение приложения, при котором все входящие соединения и фоновые операции корректно завершаются, ресурсы освобождаются, а данные сохраняются.  
> В Go 1.24.x этот паттерн особенно важен для облачных и контейнерных сред: приложения получают сигналы `SIGTERM` / `SIGINT` и должны закрываться без потери запросов или утечки goroutine.

---

## Краткий конспект

> [!summary]  
> * `signal.NotifyContext` — централизованный способ ловить сигналы ОС.  
> * `(*http.Server).Shutdown` останавливает приём новых соединений и завершает активные.  
> * Для gRPC (`grpc.Server.GracefulStop`) и worker-пулов нужен собственный порядок закрытия.  
> * Всегда задавайте **таймаут** — иначе зависшие операции затянут остановку.  
> * Закрытие должно быть **идемпотентным**: повторный вызов не должен падать.  
> * Корректный shutdown снижает утечки памяти, goroutine и дескрипторов.  
> * В Kubernetes — `preStop`, `terminationGracePeriodSeconds`, readiness-проба.

---

## Что такое graceful shutdown?

Graceful shutdown — процесс, при котором сервис перестаёт принимать новую нагрузку, но даёт текущим операциям завершиться до таймаута. В отличие от грубого «`kill -9`» мягкое выключение:

* предотвращает потерю данных (частично выполненные запросы / транзакции);  
* сохраняет метрики (сервер видит завершение, пишет логи);  
* уменьшает downtime при rolling-обновлениях.

> [!warning]  
> Если приложение игнорирует graceful shutdown, Kubernetes через `SIGKILL` оборвёт запросы и оставит повреждённые состояния.

---

## Базовый паттерн для `net/http.Server`

### 1 · Контекст сигналов

```go
ctx, stop := signal.NotifyContext(context.Background(),
    os.Interrupt, syscall.SIGTERM)
defer stop()
````

### 2 · Запуск HTTP-сервера

```go
srv := &http.Server{Addr: ":8080", Handler: mux}

go func() {
    if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
        log.Fatalf("listen: %v", err)
    }
}()
```

### 3 · Ожидание сигнала и остановка

```go
<-ctx.Done()
log.Println("shutting down…")

sdCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

if err := srv.Shutdown(sdCtx); err != nil {
    log.Printf("http shutdown: %v", err)
}
```

> [!example]  
> `http.Server.Shutdown` закрывает **listen-socket**, но даёт текущим запросам завершиться.  
> Таймаут гарантирует, что зависшие соединения не держат процесс бесконечно.

---

### Расширение паттерна через `errgroup`

```go
g, ctx := errgroup.WithContext(ctx)

// HTTP-сервер
g.Go(func() error { return srv.ListenAndServe() })

// graceful-остановка
g.Go(func() error {
    <-ctx.Done()
    sd, cancel := context.WithTimeout(context.Background(), 15*time.Second)
    defer cancel()
    return srv.Shutdown(sd)
})

if err := g.Wait(); err != nil &&
   !errors.Is(err, http.ErrServerClosed) {
    log.Fatal(err)
}
```

> [!tip]  
> `errgroup` упрощает координацию нескольких горутин и сбор ошибок при выключении.

---

## Расширенные сценарии

### gRPC-сервер

```go
grpcSrv := grpc.NewServer()
pb.RegisterGreeterServer(grpcSrv, greeter{})

g.Go(func() error {
    l, _ := net.Listen("tcp", ":9090")
    return grpcSrv.Serve(l)
})

g.Go(func() error {
    <-ctx.Done()
    grpcSrv.GracefulStop() // аналог Shutdown
    return nil
})
```

> [!example]  
> `GracefulStop` завершает RPC-стримы и закрывает соединения после отправки ответов.

---

### Worker-пул с каналом задач

```go
tasks := make(chan Job)

for i := 0; i < 8; i++ {
    go func() {
        for job := range tasks { do(job) }
    }()
}

g.Go(func() error {
    <-ctx.Done()
    close(tasks) // рабочие завершаются
    return nil
})
```

### Завершение `database/sql` запросов

```go
db, _ := sql.Open("postgres", dsn)
sdCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
defer cancel()
_ = db.Close() // эвикция пулов соединений
```

> [!warning]  
> `db.Close` **не** отменяет активные запросы. Используйте контексты с таймаутом на каждый запрос.

---

### Kafka-консьюмер (sarama)

```go
consumer := sarama.NewConsumerGroup(...)
g.Go(func() error {
    for {
        if err := consumer.Consume(ctx, topics, handler); err != nil {
            return err
        }
        if ctx.Err() != nil { return ctx.Err() }
    }
})
```

`ctx` отменится при `SIGTERM`, цикл корректно выйдет.

---

### Kubernetes — `preStop` и grace period

```yaml
lifecycle:
  preStop:
    exec:
      command: ["sh", "-c", "curl http://localhost:8080/admin/shutdown"]
terminationGracePeriodSeconds: 30
```

> [!tip]
> 
> 1. `readinessProbe` должна ставить Pod в состояние **NotReady** перед остановкой — тогда трафик не попадёт в shutting-down экземпляр.
>     
> 2. `preStop` запускается **параллельно** `SIGTERM`; приложение само ловит сигнал.
>     

---

### Docker-образ: trap SIGTERM

```dockerfile
CMD ["./server"]
STOPSIGNAL SIGTERM
```

В коде достаточно `signal.NotifyContext`.

---

## Подводные камни и типичные ошибки

|Ошибка|Последствия|Как избежать|
|---|---|---|
|Нет таймаута у `Shutdown`|Процесс «висит» навсегда|`context.WithTimeout`|
|Закрытие ресурсов в неправильном порядке|дедлоки, утечки|1) остановить ingress, 2) фоновые задачи, 3) базы|
|Нет идемпотентности|паника при повторном вызове|`sync.Once` или атомики|
|Нет логирования|сложно расследовать баги|Логируйте причины отмены и ошибки `Shutdown`|

> [!warning] **День 0 баг**  
> Забытая goroutine-producer продолжает слать задачи после закрытия канала → panic «send on closed channel». Привязывайте все goroutine к общему контексту.

---

## FAQ

> [!faq]  
> **Q:** Почему нельзя просто вызвать `ListenAndServe` в `main` и ждать сигнал?  
> **A:** Потому что `ListenAndServe` блокирует поток; для `Shutdown` нужен отдельный контроллер.
> 
> **Q:** Что произойдёт, если таймаут истечёт?  
> **A:** `Shutdown` принудительно закроет все соединения, незавершённые запросы получат ошибку.
> 
> **Q:** Нужно ли обрабатывать `SIGKILL`?  
> **A:** Нет — его невозможно перехватить; придерживайтесь `SIGTERM` / `SIGINT` и корректной grace-конфигурации.

---

## Полезные ссылки

- [https://go.dev/doc/devel/release](https://go.dev/doc/devel/release) – Release History Go
    
- [https://pkg.go.dev/net/http](https://pkg.go.dev/net/http) – пакет `net/http`
    
- [https://pkg.go.dev/os/signal](https://pkg.go.dev/os/signal) – пакет `os/signal`
    
- [https://go.dev/issue/54775](https://go.dev/issue/54775) – Issue #54775 context & shutdown
    
- [https://go.dev/doc/database/cancel-operations](https://go.dev/doc/database/cancel-operations) – Canceling in-progress operations
