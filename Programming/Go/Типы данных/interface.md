> [!info] Основная идея 
> Интерфейсы в Go предоставляют мощный и гибкий способ реализации полиморфизма, не требуя явного объявления реализации. Это облегчает композицию, делает код модульным и снижает связанность между компонентами. Понимание интерфейсов критично для эффективной работы с Go, особенно в больших проектах и при написании тестируемого кода.

## Краткий конспект

- Интерфейс в Go — это набор сигнатур методов, без реализации.
    
- Интерфейсы можно встраивать друг в друга (embedding) для композиции поведения.
    
- Реализация интерфейса происходит автоматически, если тип реализует все его методы.
    
- Интерфейсное значение под капотом хранит тип и значение (type + value).
    
- Nil-интерфейс — интерфейс без типа и значения; при вызове метода вызывает `panic`.
    
- Пустой интерфейс `interface{}` — контейнер для любого значения.
    
- Интерфейсы позволяют реализовать полиморфизм и абстракцию.
    
- Type assertion используется для извлечения значения конкретного типа.
    
- Type switch — удобный способ обрабатывать разные типы значений из одного интерфейса.
    

---

### 1. Что такое интерфейс в Go

Интерфейс — это определение поведения: он описывает набор методов, которые должен реализовать тип, чтобы соответствовать этому интерфейсу.

```go
package main

import "fmt"

type Greeter interface {
    Greet() string
}

type Person struct {
    Name string
}

func (p Person) Greet() string {
    return "Привет, " + p.Name
}

func main() {
    var g Greeter = Person{Name: "Анна"}
    fmt.Println(g.Greet())
}
```

> [!tip] В Go не нужно явно указывать, что тип реализует интерфейс — достаточно совпадения сигнатур.

---

### 2. Встраивание интерфейсов (interface embedding)

Интерфейсы можно **встраивать** друг в друга. Это позволяет создавать составные интерфейсы без дублирования методов.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader
    Writer
}
```

> [!example] Встроенный интерфейс `io.ReadWriter` включает в себя `io.Reader` и `io.Writer`.

---

### 3. Неявная имплементация: достаточно совпадения методов

Go использует **неявную реализацию**: если тип имеет методы, совпадающие с интерфейсом, он этот интерфейс реализует.

```go
type Printer interface {
    Print() string
}

type Report struct{}

func (r Report) Print() string {
    return "Печать отчёта"
}

func output(p Printer) {
    fmt.Println(p.Print())
}
```

> [!tip] Такой подход снижает связанность и облегчает внедрение зависимостей (dependency injection).

---

### 4. Устройство interface-значения

Интерфейсное значение внутри содержит **тип** конкретного значения и **указатель на само значение**. Это позволяет интерфейсу хранить значение любого типа, реализующего его методы.

```go
var i interface{} = 42
```

Внутри представляется примерно так:

```go
interface {
    type: int
    value: 42
}
```

> [!warning] Даже если `value` — `nil`, но `type` задан, интерфейс считается **не nil**.

---

### 5. Nil-интерфейс

Интерфейс считается `nil`, только если и **тип**, и **значение** равны `nil`:

```go
var i interface{} // nil
var p *Person = nil
var i2 interface{} = p // НЕ nil, т.к. тип *Person задан
```

```go
fmt.Println(i == nil)  // true
fmt.Println(i2 == nil) // false
```

> [!warning] Вызов метода на `interface{}` с `nil`-значением, но ненулевым типом, вызывает `panic`.

---

### 6. Пустой интерфейс `interface{}`

Это **универсальный контейнер** — любой тип соответствует пустому интерфейсу, так как он не требует ни одного метода.

```go
func describe(i interface{}) {
    fmt.Printf("(%v, %T)\n", i, i)
}

func main() {
    describe(42)
    describe("hello")
    describe(true)
}
```

Вывод:

```go
(42, int)
(hello, string)
(true, bool)
```

> [!example] Часто используется в JSON-декодерах, логгерах, и в API с произвольными данными.

---

### 7. Полиморфизм через интерфейсы

Интерфейсы позволяют использовать **разные типы** одинаково, если они реализуют нужные методы:

```go
type Shape interface {
    Area() float64
}

type Circle struct { Radius float64 }
func (c Circle) Area() float64 { return 3.14 * c.Radius * c.Radius }

type Square struct { Side float64 }
func (s Square) Area() float64 { return s.Side * s.Side }

func printArea(s Shape) {
    fmt.Println(s.Area())
}
```

> [!tip] Это основной способ реализации полиморфизма в Go: через интерфейсы, а не наследование.

---

### 8. Утверждение типа (type assertion)

Позволяет извлечь значение определённого типа из интерфейса.

```go
var i interface{} = "строка"

s := i.(string)       // утверждение: i — строка
fmt.Println(s)

s2, ok := i.(string)  // безопасная форма
fmt.Println(s2, ok)
```

> [!warning] Первая форма вызывает `panic`, если тип не совпадает. Вторая — безопаснее.

---

### 9. Type switch

Альтернатива множественным `type assertion` — `switch`, основанный на типе:

```go
func handle(i interface{}) {
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

> [!example] Удобно в обработке JSON, интерфейсов из внешних пакетов и универсальных коллекций.

---

## Полезные ссылки

- Tour of Go: Interfaces — официальное интерактивное введение
    
- [Effective Go: Interfaces](https://go.dev/doc/effective_go#interfaces) — подробное руководство
    
- Go Blog: The empty interface — особенности `interface{}` и отражения
    
- GeeksforGeeks: Embedding Interfaces in Go
    
- Uber Engineering: NilAway and interface panic — про `nil`-интерфейсы и безопасный код
    
- [[Интерфейсы в Go]]
    
- [[Полиморфизм без наследования]]