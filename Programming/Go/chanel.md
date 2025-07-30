## Каналы в Go: виды, назначение, чтение и запись в закрытые каналы, аксиомы каналов

### Определение и роль каналов в модели CSP

Каналы (channels) — это основной механизм взаимодействия между горутинами в Go. Они реализуют идею **обмена сообщениями вместо совместного доступа к памяти**, опираясь на концепцию **Communicating Sequential Processes (CSP)**. Канал позволяет одной горутине отправить значение, а другой — принять его, синхронизируя их без явных мьютексов.

```
ch := make(chan int)
```

Горутина, которая читает из канала, будет блокирована, пока другая не отправит в него данные, и наоборот — если канал небуферизированный.

---

## Виды каналов и их особенности

### Небуферизированные каналы

Создаются через `make(chan T)`. Они синхронны: отправка блокирует, пока не произойдёт приём.

```
ch := make(chan int)
go func() {
    ch <- 42 // блокируется, пока кто-то не прочитает
}()
fmt.Println(<-ch) // 42
```

**Плюсы**:

- Простая синхронизация
    
- Без риска переполнения буфера
    

**Минусы**:

- Возможность deadlock'а, если никто не читает
    

### Буферизированные каналы

Создаются с `make(chan T, N)`, где `N` — размер буфера. Отправка блокирует только при полном буфере.

```
ch := make(chan string, 2)
ch <- "foo"
ch <- "bar"
fmt.Println(<-ch)
fmt.Println(<-ch)
```

**Плюсы**:

- Меньше блокировок при высоких скоростях
    
- Можно использовать как очередь
    

**Минусы**:

- Риск переполнения
    
- Возможность утечки при неправильной работе
    

### Производительность

В случае высокой конкуренции, буферизированные каналы могут дать прирост в throughput за счёт асинхронности. Однако при неправильной настройке (маленький буфер) это приводит к блокировкам.

### nil‑каналы

Вот минимальный «пульт включения/выключения» канала при помощи `nil`.

```go
package main

import "fmt"

func main() {
	data := make(chan int) // рабочий канал
	ctrl := make(chan bool)

	go func() { // посылаем три числа, затем закрываемся
		for i := 1; i <= 3; i++ {
			data <- i
		}
		close(ctrl) // сигнал «хватит»
	}()

	var dch <-chan int = data // пока не nil → читаем

	for {
		select {
		case x := <-dch:
			fmt.Println("got", x)

		case <-ctrl:
			fmt.Println("stop")
			dch = nil // ▶︎ «выключаем» ветку чтения; select больше не смотрит сюда

		default:
			if dch == nil {
				return // всё закончено
			}
		}
	}
}
```

**Что здесь происходит**

- `dch` сперва указывает на реальный канал `data`, поэтому `case x := <-dch` активен.
    
- Как только приходит сигнал по `ctrl`, мы пишем `dch = nil`; с этого момента селект навсегда игнорирует ветку чтения, и код завершает работу без дополнительных флагов или `break`.

---

## Закрытые каналы: чтение и запись

### Чтение из закрытого канала

Чтение возвращает zero-value типа и `false`:

```
ch := make(chan int)
go func() {
    close(ch)
}()
val, ok := <-ch
fmt.Println(val, ok) // 0 false
```

### Запись в закрытый канал

Вызовет панику:

```
ch := make(chan int)
close(ch)
ch <- 1 // panic: send on closed channel
```

> **Правило**: **закрывать канал должен только тот, кто его создал**.

---

## Аксиомы каналов (Dave Cheney)

### 1. отправка/чтение из `nil`-канала блокирует навсегда

```go
var ch chan int
// <-ch  или ch <- 1  => блокировка навсегда
```

### 2. отправка в закрытый канал вызывает панику

```go
ch := make(chan int)
close(ch)
ch <- 42 // panic
```

### 3. чтение из закрытого канала сразу возвращает zero-value

```go
ch := make(chan int)
close(ch)
v := <-ch // 0
```

### 4. закрытие `nil`-канала вызывает панику

```go
var ch chan int
close(ch) // panic
```

---

## Примеры и паттерны

### select с каналами

```
ch1 := make(chan int)
ch2 := make(chan int)

select {
case v := <-ch1:
    fmt.Println("ch1", v)
case ch2 <- 42:
    fmt.Println("sent to ch2")
default:
    fmt.Println("nothing ready")
}
```

### fan-in

```
func fanIn(a, b <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        for {
            select {
            case v := <-a:
                out <- v
            case v := <-b:
                out <- v
            }
        }
    }()
    return out
}
```

### fan-out

```
func fanOut(in <-chan int, workers int) {
    for i := 0; i < workers; i++ {
        go func() {
            for v := range in {
                fmt.Println("worker", i, v)
            }
        }()
    }
}
```

### context + select + timeout

```
ctx, cancel := context.WithTimeout(context.Background(), time.Second)
defer cancel()

ch := make(chan int)
select {
case <-ch:
    fmt.Println("received")
case <-ctx.Done():
    fmt.Println("timeout")
}
```

### graceful shutdown с закрытием канала

```
done := make(chan struct{})

go func() {
    <-time.After(2 * time.Second)
    close(done)
}()

for {
    select {
    case <-done:
        fmt.Println("shutting down")
        return
    default:
        fmt.Println("working")
        time.Sleep(500 * time.Millisecond)
    }
}
```

---

## Best practices

- **Кто создаёт канал — тот его и закрывает**
    
- Используйте **однонаправленные каналы** для документирования интерфейсов
    
- Не используйте каналы для мутации shared state — передавайте данные, не ссылки
    
- Не полагайтесь на `close` как сигнал окончания, если не контролируете отправителя
    

---

## Частые ошибки и анти-паттерны

- **Паника при записи в закрытый канал**
    
- **Блокировка из-за nil-канала**
    
- **Отправка в канал, который никто не читает**
    
- **Закрытие канала несколькими горутинами**
    
- **Смешивание передачи и логики завершения в одном канале**
    

---

## Полезные источники

- [Channel Axioms — https://dave.cheney.net/2014/03/19/channel-axioms](https://dave.cheney.net/2014/03/19/channel-axioms)
    
- [Closing Channels — https://gobyexample.com/closing-channels](https://gobyexample.com/closing-channels)
    
- [Go Concurrency Patterns (Rob Pike) — https://www.youtube.com/watch?v=f6kdp27TYZs](https://www.youtube.com/watch?v=f6kdp27TYZs)
    
- [Go Specification — https://go.dev/ref/spec](https://go.dev/ref/spec)
    
- Understanding Go Channels — https://medium.com/@swapnildawange3650/understanding-the-behavior-of-go-channels-6aa9c3bea322
