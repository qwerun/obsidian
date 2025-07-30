> [!info] Основная идея  
> В Go конструкция **`switch`** используется для управления потоком выполнения, когда нужно выбрать один из нескольких вариантов. Это более читабельная альтернатива множественным `if-else`, а также мощный инструмент, особенно в сочетании с типами и интерфейсами.

---

## Краткий конспект

- **`switch`** — компактнее и понятнее, чем цепочка `if-else`.
- Есть несколько видов: по выражению, по типу, без условия.
- В `case` можно указывать **несколько значений**, использовать `fallthrough`.
- Часто используется с `errors.Is`, `time.Now`, `context.Err`.
- Поддерживает работу с интерфейсами через **type switch**.
- Упрощает разветвление логики в обработке ошибок, маршрутизации и CLI.

---

### Switch без выражения (`switch {}`)

Используется как альтернатива `if-else if`, особенно при сложных условиях:

```go
x := 42
switch {
case x < 0:
    fmt.Println("negative")
case x == 0:
    fmt.Println("zero")
default:
    fmt.Println("positive")
}
```

> [!example]  
> Выведет: `positive`

---

### Type switch

Позволяет проверить **тип интерфейса**:

```go
var val interface{} = 123

switch v := val.(type) {
case int:
    fmt.Printf("int: %d\n", v)
case string:
    fmt.Printf("string: %s\n", v)
default:
    fmt.Println("unknown type")
}
```

> [!example]  
> Выведет: `int: 123`

---

### Множественные значения в `case`

Можно проверять несколько значений за раз:

```go
status := 404
switch status {
case 200, 201:
    fmt.Println("OK")
case 400, 404:
    fmt.Println("Client error")
default:
    fmt.Println("Other")
}
```

---

### Директива `fallthrough`

Позволяет явно перейти к следующему `case` **без проверки условия**:

```go
switch 1 {
case 1:
    fmt.Println("one")
    fallthrough
case 2:
    fmt.Println("two")
}
```

> [!example]  
> Выведет:
> 
> ```
> one
> two
> ```

> [!warning] Ошибки
> 
> - `fallthrough` **не проверяет условие следующего `case`**, поэтому использовать его нужно с осторожностью.
>     
> - **Нет автоматического выхода** из `switch` — это Go, а не C: `break` по умолчанию есть.
>     
> - **В type switch нельзя использовать `fallthrough`**.
>     

---

### Использование с `errors.Is`, `time`, `context`

**Обработка ошибок:**

```go
switch {
case errors.Is(err, os.ErrNotExist):
    log.Println("File not found")
case errors.Is(err, os.ErrPermission):
    log.Println("Permission denied")
}
```

**Работа с временем:**

```go
now := time.Now().Weekday()
switch now {
case time.Saturday, time.Sunday:
    fmt.Println("Weekend!")
default:
    fmt.Println("Workday")
}
```

**Контекст завершения:**

```go
switch ctx.Err() {
case context.Canceled:
    log.Println("Cancelled")
case context.DeadlineExceeded:
    log.Println("Timeout")
}
```

---

## Практическое применение

### 1. Обработка HTTP-статусов

```go
func handleStatus(code int) {
    switch code {
    case 200:
        fmt.Println("OK")
    case 400, 404:
        fmt.Println("Client error")
    case 500:
        fmt.Println("Server error")
    default:
        fmt.Println("Unknown status")
    }
}
```

---

### 2. CLI-парсинг команд

```go
switch os.Args[1] {
case "help":
    fmt.Println("Usage...")
case "version":
    fmt.Println("v1.0.0")
default:
    fmt.Println("Unknown command")
}
```

---

### 3. Маршрутизация gRPC-ошибок

```go
switch status.Code(err) {
case codes.NotFound:
    return fmt.Errorf("resource not found")
case codes.Unauthenticated:
    return fmt.Errorf("login required")
default:
    return fmt.Errorf("unexpected error")
}
```

---

> [!faq] Часто задаваемые вопросы  
> **Q:** Можно ли использовать `switch` без `default`?  
> **A:** Да, это необязательный блок. Но рекомендуется добавлять `default` для надёжности.
> 
> **Q:** Можно ли использовать логические условия в `case`?  
> **A:** Да, но только в `switch {}` без выражения.
> 
> **Q:** Чем отличается `switch` от `if-else`?  
> **A:** `switch` проще читается при множественных ветках и работает быстрее в некоторых компиляциях.
> 
> **Q:** Работает ли `fallthrough` в type switch?  
> **A:** Нет, `fallthrough` разрешён только в обычных `switch`.

---

## Полезные ссылки

- [Switch - Go by Example](https://gobyexample.com/switch?utm_source=chatgpt.com)
    
- [Tour of Go — switch](https://go.dev/tour/flowcontrol/9?utm_source=chatgpt.com)
    
- [Go Specification — switch](https://go.dev/ref/spec?utm_source=chatgpt.com)
    
- [Effective Go — switch](https://go.dev/doc/effective_go?utm_source=chatgpt.com)
    
- [Go Wiki — switch](https://go.dev/wiki/Switch?utm_source=chatgpt.com)
