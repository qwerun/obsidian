# Go сегодня и микросервисы: статические (stateless) и состояниесодержащие (stateful)


---

## Краткий конспект

- **Stateless сервис:** не хранит пользовательские данные между запросами. Аналогия: киоск, который каждому покупателю выдаёт чек — вся «память» в банковской системе, а не в киоске.
    
- **Stateful сервис:** сохраняет состояние локально или требует «закрепления» клиента за конкретным экземпляром. Аналогия: ресторан со **складом** и журналом бронирований на месте.
    
- **Когда что выбирать:** по умолчанию — **stateless** (масштабировать проще). **Stateful** — только если бизнес-логика требует локального состояния (БД, очереди, хранилища).


---

## Базово о микросервисах

**Микросервис** — это маленькое приложение, решающее **одну чёткую бизнес-задачу** и общающееся по сети с другими. Примеры: сервис аутентификации, сервис платежей, сервис каталога.

> [!example]  
> Метафора: **киоск с хот-догами** против **ресторана со складом**. Киоск (stateless) не хранит продукты — каждый раз получает «снаружи». Ресторан (stateful) имеет склад, где лежат ингредиенты и история заказов.


---

## Что такое Stateless-сервис

**Определение.** Stateless-сервис **не хранит пользовательское состояние между запросами**. Любые данные — во **внешних системах**: БД, кэш (Redis), объектное хранилище, очереди.

**Свойства.** Легко масштабировать (добавляйте реплики), легко обновлять (rolling updates), требует **идемпотентности** (повторный вызов даёт тот же результат) и корректной обработки **повторов** (retry).

**Мини-пример (Go, `net/http`):** обработчик, записывающий/читающий данные через абстрактный DAO (в реальном проде — Redis/Postgres и т.п.).

```go
// main.go
package main

import (
	"context"
	"encoding/json"
	"log"
	"net/http"
	"os"
	"time"
)

// Storage — абстракция внешнего хранилища (Redis/DB и т.д.).
type Storage interface {
	Set(ctx context.Context, key string, value []byte, ttl time.Duration) error
	Get(ctx context.Context, key string) ([]byte, error)
}

type Service struct{ store Storage }

func (s *Service) create(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
	defer cancel()

	var body map[string]any
	if err := json.NewDecoder(r.Body).Decode(&body); err != nil {
		http.Error(w, "bad json", http.StatusBadRequest); return
	}
	id := r.URL.Query().Get("id")
	if id == "" { http.Error(w, "missing id", http.StatusBadRequest); return }

	data, _ := json.Marshal(body)
	if err := s.store.Set(ctx, "item:"+id, data, 10*time.Minute); err != nil {
		http.Error(w, "store error", http.StatusBadGateway); return
	}
	w.WriteHeader(http.StatusCreated)
}

func (s *Service) get(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
	defer cancel()

	id := r.URL.Query().Get("id")
	if id == "" { http.Error(w, "missing id", http.StatusBadRequest); return }

	b, err := s.store.Get(ctx, "item:"+id)
	if err != nil { http.Error(w, "not found", http.StatusNotFound); return }
	w.Header().Set("Content-Type", "application/json")
	w.Write(b)
}

func main() {
	// В реальном приложении сюда подставляется Redis/Postgres DAO.
	store := NewNoopStore() // заглушка для примера; без глобального состояния
	svc := &Service{store: store}

	mux := http.NewServeMux()
	mux.HandleFunc("POST /items", svc.create)
	mux.HandleFunc("GET /items", svc.get)

	addr := ":" + env("PORT", "8080")
	log.Println("listen", addr)
	log.Fatal(http.ListenAndServe(addr, mux))
}

// env со значением по умолчанию
func env(k, def string) string { if v := os.Getenv(k); v != "" { return v }; return def }

// NewNoopStore — простая заглушка (демо: статуса запросов, но без локального состояния).
type NoopStore struct{}
func NewNoopStore() *NoopStore { return &NoopStore{} }
func (s *NoopStore) Set(ctx context.Context, key string, v []byte, ttl time.Duration) error { return nil }
func (s *NoopStore) Get(ctx context.Context, key string) ([]byte, error) { return nil, context.Canceled }
```

Проверка:

```bash
go run main.go &
curl -X POST "http://localhost:8080/items?id=42" -d '{"name":"Alice"}' -H "Content-Type: application/json" -i
curl "http://localhost:8080/items?id=42" -i
```

**ASCII-схема (упрощённо):**

```
Client ──► LB ──► [ svc (stateless) ] x N ──► DB/Redis/Queue (внешнее состояние)
```

> [!tip]  
> Идемпотентные операции и **retry**: повтор POST может привести к дублям. Используйте идемпотентные ключи/токены, `If-Match`/версионирование записей и семантику «повтор безопасен» [[Idempotency]].

---

## Что такое Stateful-сервис

**Определение.** Stateful-сервис хранит **состояние локально** (в памяти/на диске) или требует привязки клиента к конкретной реплике (session affinity). Это характерно для БД, брокеров сообщений, кэшей в памяти, локальных загрузок файлов.

**Плюсы:** локальные операции быстрые (низкая латентность), контроль за состоянием.  
**Минусы:** сложная оркестрация (репликация, тома), масштабирование ограничено, обновления требуют осторожности.

**Пример (предостерегающий):** хранить сессии в памяти — удобно, но **не подходит** для прод-масштабирования.

```go
// main.go (anti-pattern demo)
package main

import (
	"encoding/json"
	"log"
	"net/http"
	"sync"
)

type session struct{ User string }
var (
	// Глобальное состояние: анти-паттерн для продакшена
	sessions = map[string]session{}
	mu       sync.RWMutex
)

func set(w http.ResponseWriter, r *http.Request) {
	k := r.URL.Query().Get("sid"); if k == "" { http.Error(w, "sid", 400); return }
	var s session; json.NewDecoder(r.Body).Decode(&s)
	mu.Lock(); sessions[k] = s; mu.Unlock()
	w.WriteHeader(http.StatusCreated)
}

func get(w http.ResponseWriter, r *http.Request) {
	k := r.URL.Query().Get("sid"); if k == "" { http.Error(w, "sid", 400); return }
	mu.RLock(); s, ok := sessions[k]; mu.RUnlock()
	if !ok { http.Error(w, "not found", 404); return }
	json.NewEncoder(w).Encode(s)
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("POST /sess", set)
	mux.HandleFunc("GET /sess", get)
	log.Fatal(http.ListenAndServe(":8081", mux))
}
```

Почему так **не стоит**: при масштабировании появятся несколько реплик, и «сессия» окажется только на одной. Нужны sticky-sessions/StatefulSet/внешнее хранилище — это усложняет операционку. Вместо этого храните состояние во внешней БД/Redis и делайте сервис stateless.

---

## Сравнение: Stateless vs Stateful

|Критерий|**Stateless**|**Stateful**|
|---|---|---|
|Масштабирование|Простое: добавляйте реплики, балансировщик сам распределит трафик|Сложное: репликация, шардирование, тома|
|Отказоустойчивость|Высокая: потеря реплики мало заметна|Требует восстановления состояния, возможна потеря данных|
|Обновления|Rolling-update, быстрые откаты|Нужны миграции и совместимость форматов|
|Сложность|Ниже: меньше оркестрации|Выше: стейт-машины, консенсус|
|Стоимость|Ниже на начальном этапе|Выше: хранилища, резервные копии|

**Рекомендации:** по умолчанию — **stateless**; stateful только когда необходимость подтверждена требованиями (БД, брокер, файловое хранилище, долгие сессии).

---

## Где хранить состояние правильно

**Паттерн «внешнее состояние»:** БД (PostgreSQL/MySQL), кэш (Redis), объектное хранилище (S3-совместимое), очередь сообщений (NATS/Kafka). Это позволяет держать все сервисы stateless и динамично масштабировать их.

**JWT vs серверные сессии (sticky sessions):**

- **JWT** (подписанный токен) — не требует хранилища для чтения, масштабируется легко, но важно управлять сроком жизни и отзывом токенов.
    
- **Серверные сессии** — хранятся в Redis/БД, позволяют «убивать» сессии и хранить больше данных, но добавляют операционные затраты.
    

**Мини-пример с Redis (go-redis):**

```bash
# Локально поднять Redis
docker run --rm -p 6379:6379 redis:7
```

```go
// main.go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/redis/go-redis/v9"
)

func main() {
	ctx := context.Background()
	rdb := redis.NewClient(&redis.Options{Addr: "127.0.0.1:6379"})

	if err := rdb.Set(ctx, "greet", "hello", 5*time.Minute).Err(); err != nil {
		log.Fatal(err)
	}
	val, err := rdb.Get(ctx, "greet").Result()
	if err != nil { log.Fatal(err) }
	fmt.Println("greet =", val)
}
```

> [!warning]  
> Внешние хранилища не отменяют проблемы **конкурентности**: используйте транзакции/блокировки, **идемпотентность**, разумные **таймауты** и **повторы** с джиттером.

---

## Деплой и оркестрация (очень кратко)

- **Stateless**: обычно **Kubernetes Deployment**, масштабирование репликами, без привязки к диску.
    
- **Stateful**: **Kubernetes StatefulSet** — предсказуемые имена/порядок, привязка к **PersistentVolumeClaims**; для БД/очередей это стандарт.
    
- **Готовность и живость**: `readinessProbe` и `livenessProbe` важны, чтобы трафик шёл только на здоровые реплики.
    
- **Наблюдаемость**: логи (`log/slog`), метрики (Prometheus), трассировка (OpenTelemetry) — минимальный набор для эксплуатации.
    

См. также: [[Kubernetes StatefulSet]].

---

## Тестирование и отладка

**Табличный тест для хендлера:**

```go
// handler_test.go
package main

import (
	"io"
	"net/http"
	"net/http/httptest"
	"strings"
	"testing"
)

func TestCreateValidation(t *testing.T) {
	svc := &Service{store: NewNoopStore()}
	ts := httptest.NewServer(http.HandlerFunc(svc.create))
	defer ts.Close()

	tcs := []struct{
		name, url, body string
		wantCode        int
	}{
		{"missing id", ts.URL + "/items", `{"x":1}`, http.StatusBadRequest},
		{"bad json",   ts.URL + "/items?id=1", `oops`, http.StatusBadRequest},
	}

	for _, tc := range tcs {
		resp, _ := http.Post(tc.url, "application/json", strings.NewReader(tc.body))
		if resp.StatusCode != tc.wantCode {
			b, _ := io.ReadAll(resp.Body)
			t.Fatalf("%s: got %d body=%s", tc.name, resp.StatusCode, string(b))
		}
		resp.Body.Close()
	}
}
```

**Локальные сервисы в Docker:**

```bash
# Redis
docker run --rm -p 6379:6379 redis:7
# PostgreSQL
docker run --rm -e POSTGRES_PASSWORD=pass -p 5432:5432 postgres:16
```

> [!tip]  
> Конфигурацию берите из переменных окружения: `os.Getenv("REDIS_ADDR")`, `os.Getenv("DATABASE_URL")`. Так сервис легко запускать локально, в CI и в проде.

---

## Частые ошибки новичков

1. **Хранение состояния в памяти сервиса.**  
    _Правильно:_ вынести в БД/Redis/объектное хранилище; сервис оставить stateless.
    
2. **Нет таймаутов и контекста.**  
    _Правильно:_ `context.WithTimeout` для внешних вызовов; глобальные таймауты на HTTP-клиентах и БД.
    
3. **Неидемпотентные операции с повторами.**  
    _Правильно:_ использовать идемпотентные ключи/условные обновления и дедупликацию сообщений.
    
4. **Отсутствие миграций БД.**  
    _Правильно:_ применяйте миграции (например, `migrate`), следите за совместимостью схем.
    
5. **Логирование без корреляции.**  
    _Правильно:_ корреляционные ID в заголовках/логах (trace/request-id).
    
6. **Глобальные синглтоны вместо DI.**  
    _Правильно:_ передавайте зависимости (DAO/клиенты) в конструкторе сервиса.
    
7. **Отсутствие readiness/liveness проб.**  
    _Правильно:_ определяйте отдельные ручки `/ready` и `/healthz`, валидируйте зависимости.
    

---

## Мини-FAQ

> [!faq]  
> **Stateful API — это плохо?**  
> Нет. Плохо — **безосновательно** держать состояние в памяти веб-сервиса. Базы данных, очереди, файловые хранилища по своей природе stateful и нужны.
> 
> **Что выбрать для сессий пользователей?**  
> Для масштабируемых систем — JWT или серверные сессии в Redis. Локальные сессии в памяти реплики — только для прототипа.
> 
> **Как масштабировать загрузки файлов?**  
> Отдавайте загрузки напрямую в объектное хранилище (S3-совместимое) через pre-signed URL; сервис остаётся stateless.
> 
> **Можно ли постепенно мигрировать stateful → stateless?**  
> Да: вынесите состояние наружу (БД/кэш/хранилище), добавьте слой DAO, затем поэтапно отключайте локальную «память».

---

## Пример: полноценный статут (stateless) с Redis DAO

```go
// main.go — минимальный пример статуса (stateless) с внешним хранилищем Redis.
package main

import (
	"context"
	"encoding/json"
	"log"
	"net/http"
	"os"
	"time"

	"github.com/redis/go-redis/v9"
)

type RedisDAO struct{ rdb *redis.Client }

func NewRedisDAO(addr string) *RedisDAO {
	return &RedisDAO{rdb: redis.NewClient(&redis.Options{Addr: addr})}
}

func (d *RedisDAO) Set(ctx context.Context, key string, value []byte, ttl time.Duration) error {
	return d.rdb.Set(ctx, key, value, ttl).Err()
}
func (d *RedisDAO) Get(ctx context.Context, key string) ([]byte, error) {
	return d.rdb.Get(ctx, key).Bytes()
}

type Service struct{ store interface {
	Set(context.Context,string,[]byte,time.Duration) error
	Get(context.Context,string) ([]byte, error)
}}

func (s *Service) create(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second); defer cancel()
	id := r.URL.Query().Get("id")
	if id == "" { http.Error(w, "missing id", 400); return }
	var body map[string]any
	if err := json.NewDecoder(r.Body).Decode(&body); err != nil { http.Error(w, "bad json", 400); return }
	b, _ := json.Marshal(body)
	if err := s.store.Set(ctx, "item:"+id, b, 30*time.Minute); err != nil { http.Error(w, "store fail", 502); return }
	w.WriteHeader(http.StatusCreated)
}

func (s *Service) get(w http.ResponseWriter, r *http.Request) {
	ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second); defer cancel()
	id := r.URL.Query().Get("id"); if id == "" { http.Error(w, "missing id", 400); return }
	b, err := s.store.Get(ctx, "item:"+id); if err != nil { http.Error(w, "not found", 404); return }
	w.Header().Set("Content-Type", "application/json"); w.Write(b)
}

func main() {
	addr := ":" + getenv("PORT", "8080")
	redisAddr := getenv("REDIS_ADDR", "127.0.0.1:6379")
	svc := &Service{store: NewRedisDAO(redisAddr)}

	mux := http.NewServeMux()
	mux.HandleFunc("POST /items", svc.create)
	mux.HandleFunc("GET /items", svc.get)

	log.Println("listen", addr, "redis", redisAddr)
	log.Fatal(http.ListenAndServe(addr, mux))
}

func getenv(k, def string) string { if v := os.Getenv(k); v != "" { return v }; return def }
```

Проверка:

```bash
# 1) Поднять Redis локально
docker run --rm -p 6379:6379 redis:7

# 2) Сборка/запуск
go mod init example.com/stateless && go get github.com/redis/go-redis/v9 && go run main.go

# 3) Запросы
curl -X POST "http://localhost:8080/items?id=7" -H "Content-Type: application/json" -d '{"name":"Bob"}' -i
curl "http://localhost:8080/items?id=7" -i
```

---

## Итоговые рекомендации

- По умолчанию стремитесь к **stateless** сервисам: проще масштабирование и деплой.
    
- Для состояния используйте **внешние** хранилища и очереди; держите сервис тонким.
    
- На Kubernetes: Deployment для stateless, StatefulSet — для БД/очередей. Следите за probe-ручками и наблюдаемостью.
    
- Обновляйте Go до **последнего патча** стабильной ветки — это вопрос безопасности и стабильности.
    

---

## Полезные ссылки

- Загрузки Go (актуальная версия/патчи): [https://go.dev/dl/](https://go.dev/dl/) ([Go](https://go.dev/dl/ "All releases - The Go Programming Language"))
    
- Go 1.24 — релиз в блоге: [https://go.dev/blog/go1.24](https://go.dev/blog/go1.24) ([Go](https://go.dev/blog/go1.24?utm_source=chatgpt.com "Go 1.24 is released!"))
    
- Go 1.24 — Release Notes: [https://tip.golang.org/doc/go1.24](https://tip.golang.org/doc/go1.24) ([Go Playground](https://tip.golang.org/doc/go1.24?utm_source=chatgpt.com "Go 1.24 Release Notes"))
    
- Документация Go: [https://go.dev/doc/](https://go.dev/doc/)
    
- 12-Factor App: [https://12factor.net/ru/](https://12factor.net/ru/)
    
- Martin Fowler — Microservices: [https://martinfowler.com/microservices/](https://martinfowler.com/microservices/)
    
- Kubernetes — Deployments/StatefulSets: [https://kubernetes.io/docs/](https://kubernetes.io/docs/)
    
- Redis docs: [https://redis.io/docs/](https://redis.io/docs/)
    
- OpenTelemetry: [https://opentelemetry.io/docs/](https://opentelemetry.io/docs/)
    
- NGINX Microservices Guide: [https://www.nginx.com/learn/microservices/](https://www.nginx.com/learn/microservices/)
