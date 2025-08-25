> [!info] **Основная идея**  
> `errgroup` — это удобная обёртка над `sync.WaitGroup`, которая позволяет запускать группу goroutine, дожидаться их завершения и автоматически обрабатывать первую возникшую ошибку. Он улучшает читаемость, управляемость и безопасность конкурентного кода.

---

## Краткий конспект

- Пакет `errgroup` из `golang.org/x/sync` — не из стандартной библиотеки.
- Автоматически собирает **первую ошибку** из всех goroutine.
- Обеспечивает **контекстную отмену** через `WithContext`.
- Удобнее и безопаснее, чем `sync.WaitGroup` в задачах с ошибками.
- Метод `SetLimit` помогает ограничить число параллельных goroutine.
- Отлично подходит для API-серверов, миграций и worker-пулов.
---

## Подробно

### Что такое `errgroup`

`errgroup.Group` — это структура, объединяющая несколько goroutine и предоставляющая простой способ:

- запустить их,
- отследить ошибки,
- дождаться завершения,
- при необходимости отменить оставшиеся по `context.Context`.

```go
import "golang.org/x/sync/errgroup"
````

---

### Как работает

#### `Group.Go(fn)`

Метод `Go` запускает функцию в отдельной goroutine и отслеживает её ошибку:

```go
var g errgroup.Group

g.Go(func() error {
    return doSomething()
})

if err := g.Wait(); err != nil {
    log.Println("ошибка:", err)
}
```

#### `Group.Wait()`

Дожидается завершения всех функций и возвращает первую ненулевую ошибку.

#### `errgroup.WithContext(ctx)`

Создаёт `*Group` и новый `context.Context`, который автоматически отменяется при первой ошибке.

```go
ctx := context.Background()
g, ctx := errgroup.WithContext(ctx)
```

> [!example]  
> Использование `WithContext` особенно полезно при запросах к нескольким внешним сервисам — как только один падает, остальные отменяются.

---

### Ограничение числа goroutine

С помощью `SetLimit(n int)` можно задать максимальное число одновременно выполняемых задач:

```go
g := new(errgroup.Group)
g.SetLimit(5) // максимум 5 активных goroutine одновременно
```

Это полезно для управления ресурсами — например, чтобы не перегрузить базу или сеть.

---

### Отмена через Context

Когда вы используете `WithContext`, важно уважать отмену внутри запускаемых функций:

```go
g, ctx := errgroup.WithContext(context.Background())

g.Go(func() error {
    req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
    resp, err := http.DefaultClient.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    // ...
    return nil
})
```

> [!tip]  
> Всегда передавайте `ctx` в дочерние вызовы — это позволяет быстро завершить ненужную работу.

---

### Сравнение с `sync.WaitGroup`

|Критерий|`sync.WaitGroup`|`errgroup.Group`|
|---|---|---|
|Обработка ошибок|❌ вручную|✅ автоматическая|
|Контекстная отмена|❌ не поддерживает|✅ через `WithContext`|
|Установка лимита|❌ нужно писать самому|✅ `SetLimit()`|
|Простота использования|Средняя|Выше|

> sync.WaitGroup часто используется для базового управления goroutine, но `errgroup` удобнее в сложных ситуациях с ошибками и отменами.

---

## Практическое применение

### 1. Параллельные HTTP-запросы

```go
urls := []string{"https://a.com", "https://b.com", "https://c.com"}
g, ctx := errgroup.WithContext(context.Background())

for _, u := range urls {
    url := u
    g.Go(func() error {
        req, _ := http.NewRequestWithContext(ctx, "GET", url, nil)
        _, err := http.DefaultClient.Do(req)
        return err
    })
}

if err := g.Wait(); err != nil {
    log.Fatal(err)
}
```

### 2. Миграции БД в несколько шардов

```go
shards := []string{"db1", "db2", "db3"}
g := new(errgroup.Group)

for _, dsn := range shards {
    d := dsn
    g.Go(func() error {
        return migrateShard(d)
    })
}

if err := g.Wait(); err != nil {
    log.Fatal("миграция прервана:", err)
}
```

### 3. Фоновые задачи в API-сервере

```go
g, ctx := errgroup.WithContext(context.Background())

g.Go(func() error {
    return serveHTTP(ctx)
})

g.Go(func() error {
    return serveMetrics(ctx)
})

g.Go(func() error {
    return watchSignals(ctx)
})

log.Fatal(g.Wait())
```

---

## Нюансы и подводные камни

> [!warning]  
> `errgroup` — мощный инструмент, но требует аккуратности.
>
>  🔁 **Необработанный `ctx.Done()`**: забыв его проверить, вы оставите «висячие» goroutine.
  >  
>  ❌ **Вложенные goroutine внутри `g.Go`** — неуправляемы и не отменяются автоматически.
  >  
> 🕳️ **Потенциальные утечки**, если `SetLimit` не используется при генерации большого числа задач.
  > 
> 📉 **Первая ошибка = остановка всех**, даже если другие задачи могли бы завершиться.

 
---

> [!tip] **Рекомендации**
> 
> - Используйте `WithContext` по умолчанию — это безопаснее.
>     
> - Учитывайте `SetLimit`, если работаете с тысячами задач.
>     
> - Не запускайте новые goroutine внутри `g.Go` — это нарушает контроль.
>     
> - Всегда обрабатывайте `ctx` внутри логики.
>     
> - `errgroup` хорошо работает вместе с [[Context]] и каналами.
>     

---

> [!faq] **FAQ**
> 
> **Q:** Это часть стандартной библиотеки?  
> **A:** Нет, `errgroup` входит в `golang.org/x/sync`.
> 
> **Q:** Почему нельзя просто использовать `sync.WaitGroup`?  
> **A:** `WaitGroup` не возвращает ошибку и не поддерживает отмену — всё надо делать вручную.
> 
> **Q:** Как получить все ошибки из `errgroup`, а не первую?  
> **A:** Никак напрямую — можно использовать канал для сбора ошибок внутри `g.Go`.
> 
> **Q:** Нужно ли вручную отменять контекст?  
> **A:** Если вы создаёте `ctx, cancel := context.WithCancel`, да — но `WithContext` из `errgroup` делает это сам.
> 
> **Q:** Подходит ли `errgroup` для работы с каналами?  
> **A:** Да, особенно когда нужно управлять их жизненным циклом через `ctx`.

---

## Полезные ссылки

- [golang.org/x/sync/errgroup](https://pkg.go.dev/golang.org/x/sync/errgroup?utm_source=chatgpt.com)
    
- [Devtrovert: Go errgroup Explained](https://blog.devtrovert.com/p/go-errgroup-you-havent-used-goroutines?utm_source=chatgpt.com)
    
- [Fullstory: Why Use errgroup.WithContext in Go](https://www.fullstory.com/blog/why-errgroup-withcontext-in-golang-server-handlers/?utm_source=chatgpt.com)
    
- [Go: Effective Go](https://go.dev/doc/effective_go)
