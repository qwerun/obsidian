> [!info] Основная идея  
> Тестирование gRPC-сервисов в Go критично для надёжности, рефакторинга и валидации бизнес-логики. На собеседованиях часто спрашивают про стратегии, [[Моки|моки]], работу с [[bufconn]] и стриминг — чтобы понять, как вы проектируете и проверяете взаимодействие в распределённых системах.

---

## Краткий конспект

- Unit-тесты быстрые, покрывают бизнес-логику.
- [[bufconn|Buffconn]] — способ протестировать gRPC без реального порта.
- Fake server — альтернатива мокам с реалистичными ответами.
- `t.Parallel()` важен для гонок и ускорения тестов.
- Стриминг требует особого внимания к контексту и таймаутам.
- Для e2e используйте `grpcurl` или реальные зависимости.
- Проверка ошибок через `status.Code(err)` — обязательна.
- `go test -cover` покажет покрытие, `-bench` — производительность.

---

## Уровни тестирования gRPC

| Уровень        | Что проверяется                                | Когда использовать            |
|----------------|-------------------------------------------------|-------------------------------|
| Unit           | Логика одного метода (без сервера)              | Почти всегда                  |
| Sub-server     | Отдельный метод через gRPC-интерфейс            | При работе с `bufconn`        |
| Интеграционные | Несколько компонентов (DB, MQ)                  | При CI или контракт-тестах    |
| E2E            | Полный стек (клиент + сервер)                   | Только при необходимости, не всегда в unit suite |

> [!tip]
> Не всегда нужен e2e. Если метод тривиальный, проще покрыть unit-тестом, особенно если нет сторонних зависимостей.

---

## Инструменты и пакеты

- `testing` — стандартная библиотека.
- `testify` — удобные ассерты.
- `google.golang.org/grpc/test/grpctest` — для тестирования gRPC-серверов.
- `net` + `bufconn` — запуск сервера в памяти.
- `golang/mock` — для мокинга зависимостей.
- `grpcurl` — CLI для e2e-тестирования gRPC.

---

## Пошаговый пример unit-теста с bufconn

```go
const bufSize = 1024 * 1024
var lis *bufconn.Listener

func init() {
	lis = bufconn.Listen(bufSize)
	s := grpc.NewServer()
	pb.RegisterMyServiceServer(s, &myFakeServer{})
	go s.Serve(lis)
}

func bufDialer(context.Context, string) (net.Conn, error) {
	return lis.Dial()
}

func TestMyServiceMethod(t *testing.T) {
	ctx := context.Background()
	conn, err := grpc.DialContext(ctx, "bufnet",
		grpc.WithContextDialer(bufDialer),
		grpc.WithInsecure())
	require.NoError(t, err)
	defer conn.Close()

	client := pb.NewMyServiceClient(conn)
	resp, err := client.DoSomething(ctx, &pb.MyRequest{})
	require.NoError(t, err)
	assert.Equal(t, "ok", resp.Status)
}
```

> [!example]
> Интервьюер может спросить: «А как ты тестировал gRPC без поднятия порта?» — покажите знание [[bufconn]].

---

## Тестирование стриминговых RPC

```go
stream, err := client.StreamStuff(ctx)
require.NoError(t, err)

err = stream.Send(&pb.Input{Value: "test"})
require.NoError(t, err)

res, err := stream.Recv()
require.NoError(t, err)
assert.Equal(t, "ok", res.Status)
```

**Ловушки:**
- Не забывайте `stream.CloseSend()` для bidi-stream.
- Добавляйте `t.Parallel()` при нескольких стримах.
- Контекст с таймаутом обязателен — иначе тест может повиснуть.
- Проверяйте `status.Code(err) == codes.DeadlineExceeded` при таймаутах.

---

> [!tip]
> **Mock vs in-memory server**  
> Моки (`gomock`) хорошо работают при тестировании логики клиента.  
> `bufconn`-серверы ближе к настоящей работе и позволяют отловить ошибки сериализации, сигнатур и стриминга.

---

## Типовые вопросы на собеседовании

**Что выбрать — мок или bufconn?**  
Если важно протестировать сериализацию и поведение сервера, выбирайте `bufconn`. Для изоляции — моки.

**Как протестировать `context.WithTimeout`?**  
Задайте короткий таймаут и проверьте, что вернулась ошибка с `codes.DeadlineExceeded`.

**Почему важно `t.Parallel()`?**  
Параллельные тесты выявляют гонки и ускоряют выполнение.

**Можно ли использовать `t.Fatal()` в горутине?**  
Нет, вызывайте `t.Errorf` и завершайте вручную — `t.Fatal` завершает тест немедленно.

**Как протестировать server-streaming?**  
Обходите `Recv()` в цикле до `io.EOF`. Следите за утечками.

**Когда не нужен e2e-тест?**  
Когда проверяемая логика не зависит от внешних компонентов.

**Что делает `grpcurl`?**  
Позволяет вызывать методы gRPC-сервера из CLI без написания клиента.

---

> [!warning]
> **Нюансы и подводные камни:**  
> - **context timeout**: без таймаута тест может висеть.  
> - **race-conditions**: `t.Parallel()` + shared mocks = гонки.  
> - **flaky-порты**: реальные e2e-тесты могут падать из-за занятых портов.  
> - **stream leaks**: не вызывается `CloseSend()` — стрим не закрывается.  
> - **deadline vs cancel**: путаница при обработке ошибок контекста.

---

> [!faq]  
> **Q:** Как измерить покрытие gRPC-тестов?  
> **A:** Используйте `go test -cover ./...` — работает и с `bufconn`, и с mock-тестами.  
>  
> **Q:** Как запускать все тесты?  
> **A:** Выполните `go test ./...` в корне проекта.  
>  
> **Q:** Когда использовать `gomock`, а не `bufconn`?  
> **A:** Когда важна точная изоляция внешних зависимостей и контроль входов/выходов.  
>  
> **Q:** Как тестировать ошибки `DeadlineExceeded`?  
> **A:** Передавайте `context.WithTimeout` и проверяйте `status.Code(err)`.

---

## Полезные ссылки

- [How to Test gRPC Servers in Go — Medium](https://medium.com/@3n0ugh/how-to-test-grpc-servers-in-go-ba90fe365a18?utm_source=chatgpt.com)
- [grpc-go/server_test.go — GitHub](https://github.com/grpc/grpc-go/blob/master/server_test.go?utm_source=chatgpt.com)
- [Testing gRPC methods — Medium](https://medium.com/@johnsiilver/testing-grpc-methods-6a8edad4159d?utm_source=chatgpt.com)
- [gRPC Go Basics — grpc.io](https://grpc.io/docs/languages/go/basics/?utm_source=chatgpt.com)

[[Тестирование и анализ качества|Статья тестирование]]