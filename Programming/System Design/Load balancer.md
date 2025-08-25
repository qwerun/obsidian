> [!info] Основная идея  
> **Load balancer (LB)** — это прокси-слой, который распределяет входящие соединения/запросы по пулу бэкендов, повышая **доступность**, **масштабируемость** и снижая **латентность**. На практике большинство LB по умолчанию используют **Round Robin** (часто c учётом весов — *Weighted Round Robin*): так работает NGINX (HTTP/TCP/UDP) при отсутствии другой директивы, а также AWS **ALB** на уровне target group по дефолту; в Envoy/Gateway дефолт — тоже RR/WRR. Это объясняется простотой и предсказуемостью алгоритма; в неоднородных нагрузках его часто заменяют на **Least Connections/Least Request**. :contentReference[oaicite:0]{index=0}

---

## Краткий конспект

- **Что такое LB:** программная/аппаратная прослойка, распределяющая трафик между несколькими экземплярами сервиса; даёт отказоустойчивость (health checks + автоматическое исключение «плохих» инстансов), горизонтальное масштабирование и сглаживание пиков.
- **L4 vs L7:**  
  - **L4** — балансировка TCP/UDP без понимания HTTP; дёшево и быстро (NAT/DSR/PROXY protocol).  
  - **L7** — балансировка HTTP/HTTPS/gRPC с разбором заголовков/путей; умнее маршрутизация, но дороже по CPU.
- **Алгоритмы:** Round Robin / Weighted RR, Least Connections / Least Request, Random / Power of Two Choices, Consistent Hash / Ring Hash / Maglev, (опционально) Least Time/EWMA.
- **Sticky-сессии:** «прилипание» клиента к серверу (cookies, IP-hash, header-hash); полезно для stateful, но снижает равномерность.
- **Health checks:** активные/пассивные проверки; **outlier detection** — исключение «флапающих» нод.
- **SSL termination:** завершение TLS на LB упрощает бэкенды, но переносит криптографическую нагрузку и влияет на метрики.
- **Connection draining:** мягкое выведение инстанса (finish inflight) при деплое/автоскейле.
- **Autoscaling:** LB + автошкалирование (HPA/ASG) требуют корректных health-проб, slow-start, лимитов на коннекты.
- **Выбор алгоритма:** стартуем с (Weighted) RR; при неоднородных запросах — Least Connections/Least Request; для кэшей/аффинити — Consistent Hash.

---

## Подробно

### Что такое лоад балансер

**LB** — посредник между клиентом и пулом серверов (**target group**/upstream). На **L4** он распределяет TCP/UDP-соединения (без анализа протокола приложений), а на **L7** — роутит HTTP/HTTPS/gRPC-запросы по правилам контента (URI/Host/заголовки, канареечные релизы, A/B). Применяется в монолитах и микросервисах, классических VM-стэках и в Kubernetes (Ingress/Service → Envoy/NGINX/HAProxy). В облаках LB — это managed-сервис (AWS ELB/ALB/NLB), умеющий cross-zone балансировку и health checks. :contentReference[oaicite:1]{index=1}

### Как это работает

- **Пулы/target-группы.** Набор бэкендов с параметрами (адреса, веса, *max_conns*, теги зоны).  
- **Health checks.** Активные (HTTP/TCP ping, *interval*/*timeout*, пороги UP/DOWN) и пассивные (ошибки/таймауты).  
- **Маршрутизация.** Алгоритм выбирает «следующий» бэкенд: RR/WRR, наименьшая загрузка, по хэшу ключа и т.д.  
- **Sticky/affinity.** Привязка клиента к серверу (cookie, ip_hash, header_hash) — уменьшает перетаскивание сессий. :contentReference[oaicite:2]{index=2}  
- **Slow-start.** Плавная «раскачка» новых/вернувшихся инстансов, чтобы их не залило трафиком.  
- **Outlier detection.** Автоматическое «заграждение» нод с высокой ошибочностью/латентностью.  
- **Connection draining.** На плановом выводе инстанса из ротации LB перестаёт назначать новые запросы, но даёт текущим завершиться.

### Алгоритмы балансировки (сравнение)

Ниже — краткий обзор с практическими рекомендациями.

**Round Robin (RR) / Weighted Round Robin (WRR).**  
Пробег по списку бэкендов по кругу; **вес** увеличивает частоту выбора конкретного хоста. Дефолт у NGINX (HTTP/TCP/UDP) и многих managed LB (ALB), у Envoy/Consul — тоже базовая политика. Плюсы — простота, предсказуемость; минусы — не учитывает текущую загрузку/время ответа. :contentReference[oaicite:3]{index=3}

**Least Connections / Least Request.**  
Выбирает хост с наименьшим числом активных соединений/запросов. Лучше при **неоднородных**, «тяжёлых» запросах, длинных keep-alive, HTTP/2 multiplexing. В Envoy/Istio есть *Least Request*, который часто обгоняет RR. :contentReference[oaicite:4]{index=4}

**Random / Power of Two Choices.**  
Простой рандом либо «двойной выбор» (выбрать 2 случайных хоста и взять менее загруженный). Последний даёт хороший баланс при минимальной телеметрии. В NGINX Plus есть *random*; «two choices» описан F5/NGINX. :contentReference[oaicite:5]{index=5}

**Consistent Hash / Ring Hash / Maglev.**  
Хэш по ключу (IP, cookie, header, userID), сохраняющий устойчивость при изменении пула. Подходит для кэшей, stateful-шардирования, аффинити. Envoy поддерживает *Ring Hash* и *Maglev* (псевдослучайная перестановка с равномерным распределением). Минус — потенциальный «горячий» ключ. :contentReference[oaicite:6]{index=6}

**(Опционально) Least Time/EWMA.**  
Выбирает по метрике времени ответа (EWMA/percentiles). В NGINX Plus — `least_time` (коммерческая фича). :contentReference[oaicite:7]{index=7}

#### Мини-таблица сравнения

| Алгоритм | Идея | Плюсы | Минусы | Когда использовать |
|---|---|---|---|---|
| **Round Robin / WRR** | По кругу; веса учитывают «мощность» | Простота, предсказуемость | Не видит загрузку | Стартовый baseline; однородные запросы; ALB/NGINX default :contentReference[oaicite:8]{index=8} |
| **Least Connections/Request** | Меньше активных коннектов/запросов | Лучше для «тяжёлых» запросов | Требует учёта метрик | Неоднородные нагрузки; HTTP/2/gRPC :contentReference[oaicite:9]{index=9} |
| **Random / Power of Two** | Случайный / из двух — менее загруженный | Хороший баланс при малых костах | Случайность ≠ оптимум | Когда нет метрик, но нужно лучше, чем RR :contentReference[oaicite:10]{index=10} |
| **Consistent Hash / Ring Hash / Maglev** | Аффинити по ключу | Сохранение кэш-локальности | «Горячие» ключи; баланс усложнён | Кэши, sticky-логика, шардирование :contentReference[oaicite:11]{index=11} |
| **Least Time/EWMA** | По latency (скользящее среднее) | Учитывает скорость | Требует измерений; в OSS ограничено | NGINX Plus; latency-чувствительные сервисы :contentReference[oaicite:12]{index=12} |

**Какой самый популярный алгоритм?**  
В большинстве реализаций **дефолт — Round Robin** (часто с поддержкой весов): это верно для NGINX (HTTP/TCP/UDP), AWS **ALB** (target group), а также для Envoy/Consul как базовой политики. Он понятен, легко объясним и интуитивен. Однако при **разной длительности** запросов (длинные коннекты, стримы, HTTP/2) **Least Connections/Least Request** обычно даёт более ровный баланс и меньшую tail-latency — и его стоит включать после базовых измерений. :contentReference[oaicite:13]{index=13}

---

### Практика и конфиги

#### NGINX (HTTP, L7)

*Дефолтный RR (ничего указывать не надо):*
```nginx
http {
  upstream app {
    server 10.0.0.11;   # RR по умолчанию; вес = 1
    server 10.0.0.12;   # можно добавить weight=2
  }

  server {
    listen 80;
    location / { proxy_pass http://app; }
  }
}
````

> [!example]  
> В NGINX **Round Robin — метод по умолчанию**; веса учитываются, если заданы у `server`. ([docs.nginx.com](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/?utm_source=chatgpt.com "HTTP Load Balancing | NGINX Documentation"))

_Least Connections:_

```nginx
upstream app_least {
  least_conn;                 # включаем алгоритм
  server 10.0.0.11 max_fails=3 fail_timeout=10s;
  server 10.0.0.12;
}

server {
  listen 80;
  location / { proxy_pass http://app_least; }
}
```

_IP-hash (простая аффинити по клиентскому IP):_

```nginx
upstream app_sticky {
  ip_hash;                    # простая sticky-сессия
  server 10.0.0.11;
  server 10.0.0.12;
}
```

> [!warning]  
> **IP-hash** ломает идеальную равномерность и плохо работает за NAT/CGNAT; для stateful лучше cookie-persistence на L7-прокси/приложении. ([Nginx](https://nginx.org/en/docs/http/load_balancing.html?utm_source=chatgpt.com "Using nginx as HTTP load balancer"))

#### NGINX (TCP/UDP, L4)

```nginx
stream {
  upstream pg {
    # RR — по умолчанию
    server 10.0.0.21:5432;
    server 10.0.0.22:5432;
  }
  server {
    listen 5432;
    proxy_pass pg;
  }
}
```

> RR — дефолт и в `stream{}` (TCP/UDP). ([docs.nginx.com](https://docs.nginx.com/nginx/admin-guide/load-balancer/tcp-udp-load-balancer/?utm_source=chatgpt.com "TCP and UDP Load Balancing | NGINX Documentation"))

#### HAProxy (L4/L7)

_Round Robin + health checks:_

```haproxy
backend app
  balance roundrobin
  option httpchk GET /healthz
  http-check expect status 200
  server s1 10.0.0.31:8080 check
  server s2 10.0.0.32:8080 check
```

_Least Connections:_

```haproxy
backend app_slow
  balance leastconn
  server s1 10.0.0.41:8080 check maxconn 500
  server s2 10.0.0.42:8080 check maxconn 500
```

> [!tip]  
> В HAProxy `balance roundrobin` — классика, `balance leastconn` лучше для долгих keep-alive/HTTP-стримов. Документация даёт прямые примеры синтаксиса. ([HAProxy Technologies](https://www.haproxy.com/documentation/haproxy-configuration-manual/latest/ "HAProxy Enterprise Documentation  version 3.1r1 (1.0.0-347.449) - Configuration Manual"))

#### (Опционально) Мини-прокси на Go: RR vs Least-Request

```go
// toy-lb.go: демонстрация RR и простого Least-Request по счётчику inflight.
type backend struct {
  url      string
  inflight int64 // атомарно
}
```

> В проде используйте готовые LB (NGINX/HAProxy/Envoy). Собственные велосипеды редки и дороги в сопровождении.

---

### Паттерны и подводные камни

> [!warning] На что напороться
> 
> - **Sticky-сессии vs stateless.** Привязка (cookie/IP/hash) ухудшает балансировку и усложняет рескейлинг. Хорошо для кэшей, плохо для горизонтально масштабируемых stateless-воркеров. ([Nginx](https://nginx.org/en/docs/http/load_balancing.html?utm_source=chatgpt.com "Using nginx as HTTP load balancer"))
>     
> - **Сессии/кэш без репликации.** При `ip_hash/consistent hashing` данные могут «застревать» на одном сервере; при out-of-rotation клиент попадёт на «холодную» ноду.
>     
> - **TLS termination на LB.** Легко сломать метрики: различайте _frontend/server_ latency, включайте `X-Forwarded-Proto/For` и коррелируйте на APM.
>     
> - **HTTP/2, gRPC, WebSockets.** Мультиплексирование и долгоживущие коннекты искажают RR; для HTTP/2 лучше Least Request/коннект-лимиты. ([Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancing?utm_source=chatgpt.com "Load Balancing — envoy 1.36.0-dev-61ad53 documentation"))
>     
> - **Зональность и locality.** При выключенном cross-zone (NLB) узел шлёт трафик только в своей AZ — может приводить к перекосу; включайте cross-zone, если важна ровность. ([AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html?utm_source=chatgpt.com "Network Load Balancers"))
>     
> - **Connection draining.** Не выводите инстансы «жёстко» — включайте drain/slow-start, иначе хвостовые задержки вырастут.
>     

> [!tip] Практические рекомендации
> 
> - Начинайте с **(Weighted) Round Robin** в качестве **baseline**, сразу снимайте SLO-метрики (**p95/p99** latency, RPS, 5xx, активные коннекты). ([docs.nginx.com](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/?utm_source=chatgpt.com "HTTP Load Balancing | NGINX Documentation"))
>     
> - Если нагрузка неоднородная — включайте **Least Connections/Least Request** (особенно для HTTP/2/gRPC) и измеряйте. ([Istio](https://istio.io/latest/docs/reference/config/networking/destination-rule/?utm_source=chatgpt.com "Destination Rule"))
>     
> - Для кэшей/аффинити — **Consistent Hash / Ring Hash / Maglev** (устойчивость ключей при изменении пула). ([Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancing?utm_source=chatgpt.com "Load Balancing — envoy 1.36.0-dev-61ad53 documentation"))
>     
> - Всегда включайте **health checks**, **outlier detection** и **slow-start**; это дешевле, чем реагировать на инциденты.
>     
> - Учитывайте **AZ-баланс** (cross-zone), лимиты коннектов на бэкенды и connection reuse на клиентах. ([AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html?utm_source=chatgpt.com "Network Load Balancers"))
>     

---

> [!faq] Частые вопросы
> **L4 vs L7 — что выбрать?**  
> Для «прямой трубы» (TCP/UDP, БД, TLS passthrough) — **L4**; для маршрутизации по путям/заголовкам, канареек и rate-limiting — **L7**.
>
> **Нужны ли sticky-сессии?**  
> Только если храните состояние локально (in-memory cache, сессии без внешнего хранилища). В остальных случаях избегайте — хуже баланс/масштабирование. 
>
> **Как тестировать алгоритмы?**  
> Реплеем реального трафика (WRK/K6/FortiO), сравнивайте распределение, p95/p99 и серверные метрики. Для Envoy/NGINX — прометеевские экспортеры (`active_conns`, `upstream_rq_pending_total` и пр.).
>
> **Как влияет HTTP/2/gRPC?**  
> Мультиплексирование и долгоживущие стримы делают RR менее равномерным — используйте **Least Request** и лимиты per-conn. 
>
> **Когда хватит DNS Round Robin?**  
> Только для простых статичных сервисов без health checks/дренинга; нет контроля за доступностью и реальным распределением.
>
> **ALB/NLB — чем отличаются?**  
> **ALB** — L7, дефолтный алгоритм **round robin** (можно LOR — Least Outstanding Requests); **NLB** — L4, по умолчанию балансирует внутри своей AZ, включайте cross-zone при необходимости.
>
> **Можно ли смешивать веса и least-алгоритмы?**  
> Да (Envoy: *Weighted Least Request*); но сначала убедитесь, что веса отражают реальную «мощность» инстансов.

---

## Полезные ссылки

- **NGINX — HTTP Load Balancing (алгоритмы, дефолт RR/веса):** [https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/) ([docs.nginx.com](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/?utm_source=chatgpt.com "HTTP Load Balancing | NGINX Documentation"))
    
- **NGINX Open Source: `ngx_http_upstream_module` (ip_hash/least_conn/least_time):** [https://nginx.org/en/docs/http/ngx_http_upstream_module.html](https://nginx.org/en/docs/http/ngx_http_upstream_module.html) ([Nginx](https://nginx.org/en/docs/http/ngx_http_upstream_module.html?utm_source=chatgpt.com "Module ngx_http_upstream_module"))
    
- **NGINX — TCP/UDP Load Balancing (RR по умолчанию в `stream{}`):** [https://docs.nginx.com/nginx/admin-guide/load-balancer/tcp-udp-load-balancer/](https://docs.nginx.com/nginx/admin-guide/load-balancer/tcp-udp-load-balancer/) ([docs.nginx.com](https://docs.nginx.com/nginx/admin-guide/load-balancer/tcp-udp-load-balancer/?utm_source=chatgpt.com "TCP and UDP Load Balancing | NGINX Documentation"))
    
- **HAProxy Configuration Manual (balance roundrobin/leastconn):** [https://www.haproxy.com/documentation/haproxy-configuration-manual/latest/](https://www.haproxy.com/documentation/haproxy-configuration-manual/latest/) ([HAProxy Technologies](https://www.haproxy.com/documentation/haproxy-configuration-manual/latest/ "HAProxy Enterprise Documentation  version 3.1r1 (1.0.0-347.449) - Configuration Manual"))
    
- **Envoy — Supported Load Balancers (WRR/Least Request/Ring Hash/Maglev):** [https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers) ([Envoy Proxy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancing?utm_source=chatgpt.com "Load Balancing — envoy 1.36.0-dev-61ad53 documentation"))
    
- **AWS ALB — Routing algorithms (default RR, LOR/Weighted Random):** [https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html) ([AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html?utm_source=chatgpt.com "Target groups for your Application Load Balancers"))
    
- **AWS ELB/NLB — Cross-zone load balancing:** [https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html) ([AWS Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html?utm_source=chatgpt.com "Network Load Balancers"))
    
- **NGINX — Power of Two Choices (F5 blog):** [https://www.f5.com/company/blog/nginx/nginx-power-of-two-choices-load-balancing-algorithm](https://www.f5.com/company/blog/nginx/nginx-power-of-two-choices-load-balancing-algorithm) ([F5](https://www.f5.com/company/blog/nginx/nginx-power-of-two-choices-load-balancing-algorithm?utm_source=chatgpt.com "NGINX and the \"Power of Two Choices\" Load-Balancing ..."))
    
- **NGINX OSS — общий гайд по HTTP LB:** [https://nginx.org/en/docs/http/load_balancing.html](https://nginx.org/en/docs/http/load_balancing.html) ([Nginx](https://nginx.org/en/docs/http/load_balancing.html?utm_source=chatgpt.com "Using nginx as HTTP load balancer"))
  