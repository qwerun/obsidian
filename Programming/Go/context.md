> [!info]  
> **Пакет `context`** — стандартный механизм для отмены, таймаутов и передачи сквозных метаданных (Go ≥ 1.7).  
> В релизе **Go 1.24.5** (8 июля 2025) `context` окончательно стал «шиной» между HTTP-сервером, БД-драйверами, брокерами и любым goroutine-кодом.

---

## Краткий конспект

- **Назначение** — сигнал отмены, дедлайн, таймаут и невидимые данные по цепочке вызовов.  
- Каждый объект **immutable**, порождается от родителя: root → child.  
- Базовые конструкторы: `Background`, `TODO`, `WithCancel`, `WithTimeout`, `WithDeadline`, `WithValue`.  
- Отмена — канал `<-ctx.Done()`, причина — `ctx.Err()`.  
- **Не хранить большие или изменяемые данные** в `WithValue`.  
- Создал ⇒ **обязан вызвать** `cancel()` (корни `Background/TODO` исключение).  
- Диагностика: `go vet -tags=dev -checks=cancelctx,copylocks`, pprof, runtime/trace.

---

## Подробно

### Что такое `context`

Контекст несёт три типа информации:

| Сигнал            | Метод(ы)                    | Тип            | Пример                                |
|-------------------|-----------------------------|----------------|---------------------------------------|
| Отмена            | `Done()`                    | `<-chan struct{}` | остановить HTTP-запрос при разрыве соединения |
| Таймаут/дедлайн    | `Deadline()` / `Err()`      | `time.Time` / `error` | прервать запрос к БД через 500 мс |
| Значения          | `Value(key)`                | `any`          | `request-id`, `user-id` (immutable) |

> [!tip]  
> `context` **проходной**: функция не обязана знать все поля — достаточно реагировать на `Done()`.

---

### Базовые API

| Функция                    | Назначение                                 | Нужно `cancel()` |
|----------------------------|---------------------------------------------|------------------|
| `context.Background()`     | Корень, никогда не отменяется              | ❌ |
| `context.TODO()`           | Временная заглушка                         | ❌ |
| `context.WithCancel()`     | Ручная отмена потомков                     | ✅ |
| `context.WithTimeout()`    | Таймаут + ручная отмена                    | ✅ |
| `context.WithDeadline()`   | Дедлайн + ручная отмена                    | ✅ |
| `context.WithValue()`      | Ключ-значение (request-scoped)             | ⚠️ |

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

select {
case <-time.After(3 * time.Second):
    fmt.Println("work done")
case <-ctx.Done():
    log.Fatal(ctx.Err()) // deadline exceeded
}
````

---

### Практическое применение

#### HTTP-обработчики

```go
func handler(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()                                  // таймаут сервера
    result, err := db.QueryContext(ctx, "SELECT ...")   // прерывается автоматически
    ...
}
```

#### Запросы к БД

```go
ctx, cancel := context.WithDeadline(ctx, time.Now().Add(500*time.Millisecond))
defer cancel()

if err := tx.CommitContext(ctx); err != nil {
    return fmt.Errorf("commit: %w", err)
}
```

#### Goroutine-fan-out (`errgroup`)

```go
g, ctx := errgroup.WithContext(context.Background())

for _, url := range urls {
    url := url
    g.Go(func() error {
        reqCtx, cancel := context.WithTimeout(ctx, 1*time.Second)
        defer cancel()
        return fetch(reqCtx, url)
    })
}
if err := g.Wait(); err != nil { log.Fatal(err) }
```

---

### Лучшие практики

> [!tip] **Чек-лист**
> 
> 1. Создал → `defer cancel()` (кроме `Background`, `TODO`).
>     
> 2. В `WithValue` только **immutable-типы** (string, int64, UUID).
>     
> 3. `context` — **первый аргумент** функции.
>     
> 4. Не сохраняйте `ctx` в структурах — передавайте по стеку вызовов.
>     
> 5. Проверяйте `<-ctx.Done()` в долгих циклах и I/O-лупах.
>     

---

### Нюансы и подводные камни

> [!warning]
> 
> - **Забытый `cancel()`** → утечка таймера и goroutine.
>     
> - **WithValue вместо параметров** усложняет тесты и замедляет вызовы.
>     
> - **Старый контекст** для фоновой задачи — создайте новый root.
>     
> - Большие объекты в `WithValue` дороги: `Value` ищет по всей цепочке.
>     
> - Shadowing (`ctx := context.TODO()`) рвёт цепочку отмены.
>     

|До (антипаттерн)|После (исправлено)|
|---|---|
|`go<br>ctx := context.Background()<br>ctx, _ = context.WithTimeout(ctx, t)<br>// cancel забыли`|`go<br>ctx, cancel := context.WithTimeout(context.Background(), t)<br>defer cancel()`|
|`go<br>userID := ctx.Value("user")`|`go<br>type userKey struct{}<br>ctx = context.WithValue(ctx, userKey{}, id)`|

---

### FAQ

> [!faq]  
> **Q:** Можно ли отменить контекст внутри своей goroutine?  
> **A:** Да, владелец обязан вызвать `cancel()` при завершении.  
> 
> **Q:** Что вернёт `ctx.Err()` для `Background()`?  
> **A:** Всегда `nil` — он не отменяется.  
> 
> **Q:** Почему не стоит хранить логгер в контексте?  
> **A:** Легче прокинуть его зависимостью; `WithValue` для request-данных.  
> 
> **Q:** Как сделать «бесконечный» таймаут?  
> **A:** Используйте `WithCancel` и вызывайте `cancel()` только при необходимости.  
> 
> **Q:** Показывает ли trace отмену?  
> **A:** Да, события `block/select/ctx` и `unblock` видны в `go tool trace`.

---

## Полезные ссылки

- [https://pkg.go.dev/context](https://pkg.go.dev/context)
    
- [https://go.dev/blog/context](https://go.dev/blog/context)
    
- [https://go.dev/doc/devel/release](https://go.dev/doc/devel/release)
    
- [https://github.com/golang/go/wiki/Context](https://github.com/golang/go/wiki/Context)
