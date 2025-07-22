**`bufconn`** — это пакет `google.golang.org/grpc/test/bufconn`, дающий **in‑memory `net.Listener`**.

- Позволяет поднять gRPC‑сервер и подключиться к нему клиентом **без открытия TCP‑порта**: данные идут через буфер в памяти.
    
- Используется в юнит‑тестах для скорости, детерминированности и отсутствия конфликтов по портам.
    
- Шаги:
    
    1. `listener := bufconn.Listen(bufSize)`
        
    2. `grpcServer.Serve(listener)` (в goroutine)
        
    3. Клиенту при `grpc.DialContext` передать dialer, который делает `listener.Dial()`.
        

Итог: быстрый, изолированный способ тестировать gRPC‑сервисы внутри одного процесса.