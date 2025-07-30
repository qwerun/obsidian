> [!info]  
> В Go константа `nil` получает тип из контекста выражения — от указателя до канала. Такое **типизированное nil** приводит к неожиданным багам: значение «пустое», но динамический тип присутствует. Правильная проверка и понимание устройства интерфейса критичны для production-кода.

## Почему typed-nil опасен

Внутри **пустого интерфейса** (`interface{}`) хранятся два слова: указатель на таблицу методов (тип) и указатель на данные (значение). Если первое слово не `nil`, то весь интерфейс уже **не равен `nil`**, даже если второе слово — нулевой указатель.

```go
var p *int        // истинный nil-указатель
var i interface{} // nil-интерфейс

i = p              // теперь i != nil
fmt.Println(i)     // <nil>
````

То же случается при упаковке нулевых срезов, карт и каналов:

```go
var s []int        // nil-срез
var m map[string]int
var ch chan int

fmt.Println(s == nil) // true
var j any = s
fmt.Println(j == nil) // false — динамический тип []int уже установлен
```

## Классическая ловушка с `error`

```go
type myErr struct{ msg string }

func (e *myErr) Error() string { return e.msg }

func mayFail(b bool) error {
    if b {
        var zero *myErr                // nil-пойнтер
        return zero                    // ОПАСНО: typed-nil
    }
    return nil
}

if err := mayFail(true); err != nil { // err != nil, хотя zero-пойнтер
    log.Println("unexpected:", err)
}
```

> [!warning]  
> Частый баг: возвращать typed-nil-значение, завернутое в `error` интерфейс. Проверка `err != nil` ложно срабатывает, что ломает логику обработки ошибок.

### Корректная проверка

_Возвращайте_ «истинный» nil:

```go
if b { return (*myErr)(nil) } // явная типизация и nil-значение
```

_Либо_ извлекайте указатель из интерфейса и проверяйте его:

```go
if e, ok := err.(*myErr); ok && e == nil {
    // typed-nil detected
}
```

## nil в контейнерах

|Тип|При сравнении|При упаковке в `interface{}`|
|---|---|---|
|`[]T`|`s == nil` OK|`any(s) != nil`|
|`map[K]V`|`m == nil` OK|`any(m) != nil`|
|`chan T`|`c == nil` OK|`any(c) != nil`|
|`func()`|`f == nil` OK|`any(f) != nil`|

Рефлект-пакет умеет отличать typed-nil:

```go
if v := reflect.ValueOf(x); v.Kind() == reflect.Pointer && v.IsNil() {
    // истинный нулевой указатель
}
```

## Больше примеров

### 1. Ошибка в обработчике HTTP

```go
func handler(w http.ResponseWriter, r *http.Request) error {
    var p *Payload // decode ошибся — p == nil
    return p       // НЕЛЬЗЯ
}

if err := handler(w, r); err != nil {
    // всегда истинно ⇢ 500 вместо 204
}
```

### 2. Ленивая инициализация через `sync.Once`

```go
var once sync.Once
var global *Config

func Cfg() *Config {
    once.Do(func() { global = load() })
    return global // может быть typed-nil, если load вернул nil
}

cfg := Cfg()
if cfg == nil { /* ... */ } // безопасно: указатель, не интерфейс
```

### 3. Канал-фантом

```go
var ch chan int
select {            // блокируется навсегда
case <-ch:          // panic: send/recv on nil channel
default:
}
```

Поскольку `ch == nil`, операция чтения/записи на нём блокируется навечно — используйте проверки или инициализируйте пустой буферизированный канал.

## Практические советы

1. **Пишите юнит-тесты с `nil`**: для каждого API-метода добавляйте кейс «нулевой вход».
    
2. **Не возвращайте typed-nil**: всегда документируйте, какой тип ошибки — и возвращайте `nil` в явном виде (`(*MyErr)(nil)`).
    
3. **Соблюдайте zero-value-friendly API**: функции обязаны корректно работать с нулём без паники.
    
4. **Используйте вспомогательные конструкторы**:
    
    ```go
    func New[T any](v *T) *T { return v } // гарантирует nil-safety
    ```
    
5. **Статический анализ**: подключите `staticcheck SA5004` — он ловит typed-nil для ошибок.
