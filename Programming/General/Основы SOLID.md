## Основная идея

> [!info]
> **SOLID** — пять принципов объектно-ориентированного проектирования.  
> В Go, несмотря на отсутствие классов/наследования, принципы применяются через **интерфейсы** и **композицию структур** и помогают держать код читабельным, тестируемым и расширяемым.

**Расшифровка SOLID**:

- **S — SRP (Single Responsibility Principle)** — _принцип единственной ответственности_
- **O — OCP (Open–Closed Principle)** — _принцип открытости/закрытости_
- **L — LSP (Liskov Substitution Principle)** — _принцип подстановки Лисков_
- **I — ISP (Interface Segregation Principle)** — _принцип разделения интерфейса_
- **D — DIP (Dependency Inversion Principle)** — _принцип инверсии зависимостей_

---

## Краткий конспект

- **SOLID-принципы** — SRP, OCP, LSP, ISP, DIP.  
- **Преимущества** — модульность, низкая связанность, высокая переиспользуемость.  
- **Практика** — рефакторинг, микросервисный дизайн.

---

## Подробно

### Что такое SOLID?

Термин предложил Роберт К. Мартин («Uncle Bob»). В Go принципы реализуют через **малые интерфейсы**, **композицию** и «приём: _принимаем интерфейсы — возвращаем структуры_».

---

#### S — Single Responsibility

_У сущности должна быть одна причина для изменения._

```go
// Плохо: смешаны генерация и запись на диск.
type Report struct{ Data []byte }

func (r *Report) Generate() []byte               { /* ... */ return nil }
func (r *Report) Save(path string) error         { /* ... */ return nil } // файловая логика внутри

// Хорошо: обязанности разнесены.
type ReportGenerator interface { Generate() []byte }
type FileSaver       interface { Save([]byte, string) error }

type ReportService struct {
	gen ReportGenerator
	fs  FileSaver
}

func (s ReportService) Export(path string) error {
	return s.fs.Save(s.gen.Generate(), path)
}
````

_Изменение формата отчёта не затрагивает сохранение и наоборот._

---

#### O — Open–Closed

_«Открыт для расширения, закрыт для изменения»._

```go
// До: добавление формата требует менять switch (и пересобирать клиентов).
func Export(v any, format string) ([]byte, error) {
	switch format {
	case "json":
		return json.Marshal(v)
	case "yaml":
		return yaml.Marshal(v)
	default:
		return nil, errors.New("unknown format")
	}
}

// После: регистрируем энкодеры.
type Encoder interface{ Encode(v any) ([]byte, error) }

var registry = map[string]Encoder{} // обычно заполняется в init(); для runtime-изменений защитите мьютексом

func Register(name string, enc Encoder) { registry[name] = enc }

func Export(v any, format string) ([]byte, error) {
	enc, ok := registry[format]
	if !ok { return nil, fmt.Errorf("unknown format: %s", format) }
	return enc.Encode(v)
}
```

> [!tip]  
> В Go расширяемость часто = регистрация новых реализаций интерфейса (плагины, драйверы, стратегии) без изменения клиентской функции.

---

#### L — Liskov Substitution

если у нас есть интерфейс и несколько его реализаций, то любая реализация должна работать в местах, где ожидается этот интерфейс, **без сюрпризов

```go
package main

import "fmt"

// Базовый интерфейс
type Bird interface {
	Fly() string
}

// Реализация: Воробей умеет летать
type Sparrow struct{}

func (s Sparrow) Fly() string {
	return "Sparrow is flying"
}

// Реализация: Пингвин не умеет летать, но мы всё равно заставляем его реализовать Fly
type Penguin struct{}

func (p Penguin) Fly() string {
	return "Penguin is flying... (но на самом деле нет)"
}

// Функция, которая работает с любыми птицами
func makeBirdFly(b Bird) {
	fmt.Println(b.Fly())
}

func main() {
	makeBirdFly(Sparrow{}) // OK
	makeBirdFly(Penguin{}) // Нарушение LSP
}

```


---

#### I — Interface Segregation

_Лучше несколько маленьких интерфейсов, чем один «толстый»._

```go
// Плохо: интерфейс навязывает лишнее.
type Service interface {
	Start() error
	Stop() error
	Reload() error
	Status() (int, error)
}

// Хорошо: клиенты зависят только от нужного поведения.
type Starter interface{ Start() error }
type Stopper interface{ Stop() error }
type Reloader interface{ Reload() error }
```

> [!tip]  
> В Go применяйте «микро-интерфейсы» (напр., `io.Reader`, `io.Writer`) и **вводите интерфейс в пакете потребителя**, а не в пакете реализации.

---

#### D — Dependency Inversion

_Зависимости направлены к абстракциям, а не к деталям._

```go
// Плохо: хардкод конкретной БД.
type UserRepo struct{ db *sql.DB }

// Хорошо: репозиторий зависит от абстракции, которая определена у потребителя.
type DB interface {
	QueryRowContext(ctx context.Context, q string, args ...any) *sql.Row
}
type UserRepo struct{ db DB }

func NewUserRepo(db DB) UserRepo { return UserRepo{db: db} }
```

> [!tip]  
> Не плодите «интерфейс-ради-интерфейса». Вводите его там, где есть **несколько** возможных реализаций **или** нужна изоляция для тестов.

---

### Нюансы и подводные камни

> [!warning]
> 
> - **Переинжениринг:** избыточные слои усложняют код.
>     
> - **Абстракция ради абстракции:** лишние интерфейсы ухудшают производительность и читаемость.
>     
> - **Идиомы Go:** избегайте «геттеров/сеттеров» без необходимости; предпочитайте явные функции и публичные поля там, где это уместно.
>     

---

### Рекомендации и FAQ

> [!tip]  
> Начинайте с **SRP** и **ISP**: обычно дают максимум пользы за минимум усилий. Объединяйте с «accept interfaces, return structs».

> [!faq]  
> **Q:** Есть ли инструменты, проверяющие SOLID в Go?  
> **A:** Прямых «SOLID-линтеров» нет, но помогут `go vet`, `staticcheck`, `gocyclo`, `gosec`, а также покрытие тестами.
> 
> **Q:** Как внедрить DIP без DI-контейнера?  
> **A:** Фабрики и явная передача зависимостей в конструктор (`New(ServiceDeps)`); интерфейсы — в пакете потребителя; моки — генераторами.

---

## Полезные ссылки

- [https://golang.org/doc/effective_go](https://golang.org/doc/effective_go)
    
- Репозитории с примерами SOLID на Go
    
- Книги Роберта Мартина: «Чистый код», «Принципы, паттерны и методики Agile-разработки ПО»