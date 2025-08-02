## Основная идея

> [!info]  
> **SOLID** — пять принципов объектно-ориентированного проектирования.  
> Эти правила помогают разработчикам Go писать код, который легче читать, сопровождать, тестировать и масштабировать, даже несмотря на отсутствие классов — достаточно интерфейсов и композиции структур.

Расшифровка аббревиатуры **SOLID**:

- **  
    _Принцип единственной ответственности_
    
- **O — OCP (Open-Closed Principle)**  
    _Принцип открытости/закрытости_
    
- **L — LSP (Liskov Substitution Principle)**  
    _Принцип подстановки Лисков_
    
- **I — ISP (Interface Segregation Principle)**  
    _Принцип разделения интерфейса_
    
- **D — DIP (Dependency Inversion Principle)**  
    _Принцип инверсии зависимостей_
---

## Краткий конспект

- **SOLID-принципы** — SRP, OCP, LSP, ISP, DIP.
    
- **Преимущества**: модульность, низкая связанность, высокая переиспользуемость.
    
- **Практическое применение**: рефакторинг, микросервисный дизайн.
    

---

## Подробно

### Что такое SOLID?

Термин ввёл Роберт К. Мартин («Uncle Bob») в начале 2000-х как набор практических правил для ООП. В Go отсутствуют классы и наследование в привычном виде, но композиция структур и интерфейсов позволяет применять SOLID не менее эффективно: каждый принцип трансформируется в «композиционный стиль» Go.

### Расшифровка и примеры

#### **S — Single Responsibility**

_Каждый модуль должен иметь лишь одну причину для изменения._

```go
type Report struct {
	Data []byte
}

func (r *Report) Generate() []byte { /* ... */ }
func (r *Report) Save(path string) error { /* ... */ } // + файловая логика
```

```go
type ReportGenerator interface {
	Generate() []byte
}

type FileSaver interface {
	Save([]byte, string) error
}
```

_Разделив генерацию и сохранение, мы получили два независимых изменения: формат отчёта и способ хранения._

---

#### **O — Open-Closed** 

_Код должен быть открыт для расширения, но закрыт для изменения._

```go
// До: каждый новый экспорт требует правки switch.
func Export(data []byte, format string) ([]byte, error) {
	switch format {
	case "json":
		return json.Marshal(data)
	case "yaml":
		return yaml.Marshal(data)
	}
	return nil, errors.New("unknown format")
}

// После: вводим интерфейс и регистрируем расширения без правки функции.
type Encoder interface {
	Encode(v any) ([]byte, error)
}

var registry = map[string]Encoder{}

func Register(name string, enc Encoder) { registry[name] = enc }

func Export(data any, format string) ([]byte, error) {
	enc, ok := registry[format]
	if !ok {
		return nil, errors.New("unknown format")
	}
	return enc.Encode(data)
}
```

_Новые форматы добавляются регистрацией нового `Encoder`, не меняя `Export`._

---

#### **L — Liskov Substitution**  

_Подтипы должны полностью заменять базовые типы без нарушения логики._

```go
type Reader interface {
	Read() ([]byte, error)
}

type FileReader struct{/* ... */}
func (f *FileReader) Read() ([]byte, error){/* ... */}

type CachingReader struct {
	r Reader
}
func (c *CachingReader) Read()([]byte,error){/* ... */}
```

_Любой `Reader` можно подставить вместо другого, не ломая клиентский код._

---

#### **I — Interface Segregation**  

_Лучше несколько специализированных интерфейсов, чем один «толстый»._

```go
// Плохо: огромный интерфейс.
type Service interface {
	Start() error
	Stop() error
	Reload() error
	Status() (int, error)
}

// Хорошо: разбиваем.
type Starter interface{ Start() error }
type Stopper interface{ Stop() error }
type Reloader interface{ Reload() error }
```

_Клиент зависит только от нужного ему поведения._

---

#### **D — Dependency Inversion** 

_Модули верхнего уровня не должны зависеть от конкретных реализаций._

```go
// Неудачное внедрение: прямой импорт.
type UserRepo struct { db *sql.DB }

// Улучшено с DI через интерфейс.
type DB interface {
	QueryRow(string, ...any) *sql.Row
}
type UserRepo struct { db DB }
```

_`UserRepo` требует абстракцию `DB`, а конкретный `*sql.DB` вводится извне._
### Нюансы и подводные камни

> [!warning]
> 
> - **Переинжениринг** — слишком много слоёв добавляет лишнюю сложность.
>     
> - **Абстракция ради абстракции** может снизить производительность.
>     
> - **Идиомы Go**: избегайте «Getter/Setter»-паттернов; предпочитайте простые функции.
>     

### Рекомендации и FAQ

> [!tip]  
> Начинайте с SRP и ISP — они дают наибольшую отдачу при минимальных затратах.

> [!faq]  
> **Q:** Есть ли инструменты, проверяющие SOLID для Go?  
> **A:** Прямых нет, но [[Тестирование и анализ качества#go vet|go vet]], `staticcheck`, `gocyclo`, `gosec` помогут выявить нарушенную связанность.
> 
> **Q:** Как внедрить DIP без DI-контейнера?  
> **A:** Используйте фабричные функции и вручную прокидывайте зависимости через параметры конструктора.

---

## Полезные ссылки

- [https://golang.org/doc/effective_go](https://golang.org/doc/effective_go)
    
- GitHub-репозитории с примерами SOLID на Go
    
- Книги Роберта Мартина («Чистый код», «Принципы, паттерны и методики Agile-разработки ПО»)