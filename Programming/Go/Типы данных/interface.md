> [!info] Основная идея 
> Интерфейсы в Go — это способ описать **поведение** через набор сигнатур методов. Реализация интерфейса происходит **неявно** (structural typing): если тип имеет все нужные методы — он «подходит». Это упрощает композицию, снижает связанность и делает код тестируемым.

## Краткий конспект

- Интерфейс — **набор сигнатур методов** (без реализации).
- Интерфейсы можно **встраивать** друг в друга (embedding) для композиции поведения.
- Реализация интерфейса происходит **автоматически**, если тип реализует все его методы.
- Интерфейсное значение хранит **динамический тип** и **данные** (в рантайме: `type + data`; для не-пустых интерфейсов — через `itab + data`).
- Интерфейс равен `nil`, **только если и тип, и значение равны nil**. Если тип есть, а значение `nil` (например, `(*T)(nil)`), то **интерфейс не nil**.
- Пустой интерфейс `interface{}` (синоним `any` с Go 1.18) принимает **любое** значение.
- Полиморфизм в Go достигается через **интерфейсы**, а не наследование.
- `type assertion` и `type switch` — способы работать с конкретными типами внутри интерфейса.
- 🔎 Важно: **набор методов (method set)** зависит от приёмника:  
  — метод с приёмником `T` присутствует у `T` и `*T`;  
  — метод с приёмником `*T` присутствует **только** у `*T`.  
  Поэтому иногда **`*T` реализует интерфейс, а `T` — нет**.

---

### 1. Что такое интерфейс в Go

Интерфейс описывает **контракт**: набор методов, которые должен предоставить тип.

```go
package main

import "fmt"

type Greeter interface {
    Greet() string
}

type Person struct {
    Name string
}

func (p Person) Greet() string { // value-receiver
    return "Привет, " + p.Name
}

func main() {
    var g Greeter = Person{Name: "Анна"} // неявная реализация
    fmt.Println(g.Greet())
}
````

> [!tip] Не нужно «implements»: достаточно совпадения сигнатур.

---

### 2. Встраивание интерфейсов (interface embedding)

```go
type Reader interface  { Read(p []byte) (int, error) }
type Writer interface  { Write(p []byte) (int, error) }
type ReadWriter interface {
    Reader
    Writer
}
// io.ReadWriter в стандартной библиотеке устроен так же.
```

---

### 3. Неявная имплементация

```go
type Printer interface { Print() string }

type Report struct{}
func (Report) Print() string { return "Печать отчёта" }

func output(p Printer) { fmt.Println(p.Print()) }
```

> [!tip] Интерфейс обычно **объявляет потребитель** (тот, кто вызывает методы), а не библиотека-поставщик.

---

### 4. Устройство interface-значения

На уровне идей интерфейс — это пара **(тип, данные)**.

```go
var i interface{} = 42
// conceptually:
i = {
    type:  int,
    value: 42,
}
```

> [!warning] Если **тип задан, а данные = nil** (например, `var p *T = nil; var i interface{} = p`), то `i != nil`.

---

### 5. Nil-интерфейс и вызовы методов

Интерфейс `nil` ⇢ **нет типа и значения** — любой вызов метода через такой интерфейс **panic**.

```go
var r io.Reader // r == nil
_ = r.Read(nil) // panic: nil interface has no dynamic type
```

Если же в интерфейсе лежит **тип `*T`, но само значение nil**, вызов метода **возможен**: метод получит nil-приёмник. Он может:

- безопасно обработать nil и вернуть ошибку,
    
- или упасть, если разыменует указатель.
    

```go
type S struct{}
func (s *S) M() { fmt.Println("ok with nil receiver too") }

var s *S = nil
var i interface{ M() } = s // НЕ nil
i.M() // корректный вызов; внутри M решаем, что делать с nil
```

> [!tip] Это ключевое отличие от «nil интерфейса»: у «типизированного nil» вызов возможен.

---

### 6. Пустой интерфейс `interface{}` / `any`

```go
func describe(i any) { // any == interface{}
    fmt.Printf("(%v, %T)\n", i, i)
}

describe(42)      // (42, int)
describe("hello") // (hello, string)
```

> [!example] Полезно для логгирования и «сырого» JSON, но **не злоупотребляйте**: предпочитайте конкретные типы или дженерики.

---

### 7. Полиморфизм через интерфейсы

```go
type Shape interface { Area() float64 }

type Circle struct{ R float64 }
func (c Circle) Area() float64 { return math.Pi * c.R * c.R }

type Square struct{ A float64 }
func (s Square) Area() float64 { return s.A * s.A }

func printArea(s Shape) { fmt.Println(s.Area()) }
```

---

### 8. Утверждение типа (type assertion)

```go
var i any = "строка"

s  := i.(string)      // panic, если не string
s2, ok := i.(string)  // безопасно: ok=false, если тип не совпал
_ = s; _ = s2; _ = ok
```

---

### 9. Type switch

```go
func handle(i any) {
    switch v := i.(type) {
    case int:
        fmt.Println("int:", v)
    case string:
        fmt.Println("string:", v)
    default:
        fmt.Println("unknown type")
    }
}
```

---

> [!warning] Частые ошибки
> 
> - «Почему `T` не реализует интерфейс, а `*T` реализует?» — проверьте **method set** и приёмники.
>     
> - «Почему `i == nil` ложно?» — в интерфейсе может лежать **типизированный nil**.
>     
> - «Почему вызов метода упал?» — метод мог получить **nil-приёмник** и разыменовать его.
>     

> [!tip] Практика
> 
> - Держите интерфейсы **маленькими** (обычно 1–2 метода).
>     
> - Объявляйте интерфейсы **на стороне потребителя**.
>     
> - Для обобщений предпочитайте **дженерики** вместо `interface{}`.
>     

---

## Полезные ссылки

- Tour of Go — Interfaces
    
- [Effective Go: Interfaces](https://go.dev/doc/effective_go#interfaces)
    
- Go Blog: The empty interface
    
- [[Интерфейсы в Go]]
    
- [[Полиморфизм без наследования]]