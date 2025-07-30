> [!info] Основная идея  
> `EXPLAIN` и `EXPLAIN ANALYZE` в PostgreSQL позволяют понять, **как СУБД будет выполнять ваш запрос**, какие шаги она предпримет, какие индексы задействует и насколько эффективно это произойдёт. Умение читать планы — важный навык для любого backend-разработчика, работающего с реальными данными.

---

## Краткий конспект

- `EXPLAIN` показывает **план выполнения** запроса (без его выполнения).
    
- `EXPLAIN ANALYZE` **выполняет** запрос и сравнивает фактические показатели с планируемыми.
    
- Важно понимать поля: `cost`, `rows`, `width`, `actual time`, `loops`, `buffers` и др.
    
- Планы могут различаться: `Seq Scan`, `Index Scan`, `Hash Join`, `Nested Loop` и др.
    
- Статистика таблиц влияет на выбор плана. Иногда нужно запускать `ANALYZE`.
    
- Оптимизация = читаем план → находим узкое место → правим SQL или схему.
    

---

## Подробно

### Что такое `EXPLAIN` и зачем он нужен

Когда вы пишете SQL-запрос, PostgreSQL строит для него **план выполнения**. План — это дерево операций: сканирование таблиц, соединения, фильтры, сортировки. `EXPLAIN` помогает посмотреть, **что именно собирается делать СУБД**, прежде чем запрос будет запущен.

```
EXPLAIN SELECT * FROM users;
```

Он покажет, будет ли выполнено последовательное сканирование (Seq Scan), задействован индекс, или будут другие оптимизации.

### Чем отличается `EXPLAIN ANALYZE`

Добавив `ANALYZE`, мы просим СУБД **выполнить** запрос и **замерить** реальные параметры:

```
EXPLAIN ANALYZE SELECT * FROM users;
```

Появятся `actual time`, `rows`, `loops`, `buffers`, а также итоговое `Planning Time` и `Execution Time`.

Это особенно полезно для диагностики отклонений между планом и реальностью.

---

### Расшифровка основных полей

|Поле|Что означает|
|---|---|
|`cost=..`|Оценка стоимости (внутренние единицы, относительные)|
|`rows`|Сколько строк **предполагается** вернуть|
|`width`|Средний размер строки в байтах|
|`actual time`|Фактическое время начала и конца операции (в миллисекундах)|
|`loops`|Сколько раз шаг повторился (например, при вложенных циклах)|
|`buffers`|Детали доступа к страницам (shared hit, read, dirtied и т.д.)|
|`Planning Time`|Сколько времени ушло на построение плана|
|`Execution Time`|Время фактического выполнения запроса|

---

## Практические примеры

### Пример 1: Простой `SELECT` без индекса (Seq Scan)

```
SELECT * FROM users WHERE name = 'Alice';
```

```
EXPLAIN SELECT * FROM users WHERE name = 'Alice';
```

```
Seq Scan on users  (cost=0.00..12.75 rows=1 width=36)
  Filter: (name = 'Alice')
```

**Комментарий:** PostgreSQL сканирует всю таблицу `users` в поиске совпадений. Нет индекса — значит, `Seq Scan`.

### Пример 2: После добавления индекса

```
CREATE INDEX idx_users_name ON users(name);
ANALYZE users;

EXPLAIN SELECT * FROM users WHERE name = 'Alice';
```

```
Index Scan using idx_users_name on users  (cost=0.15..8.17 rows=1 width=36)
  Index Cond: (name = 'Alice')
```

**Комментарий:** теперь используется индекс. `Index Scan` — более эффективен, особенно на больших таблицах.

---

### Пример 3: `JOIN` с Hash Join

```
SELECT * FROM users JOIN orders ON users.id = orders.user_id;
```

```
EXPLAIN SELECT * FROM users JOIN orders ON users.id = orders.user_id;
```

```
Hash Join  (cost=1.05..23.30 rows=10 width=64)
  Hash Cond: (orders.user_id = users.id)
  ->  Seq Scan on orders  (cost=0.00..11.40 rows=140 width=32)
  ->  Hash  (cost=1.00..1.00 rows=5 width=32)
        ->  Seq Scan on users  (cost=0.00..1.00 rows=5 width=32)
```

**Комментарий:** PostgreSQL сначала сканирует `users`, хеширует их по `id`, затем для каждой строки из `orders` ищет совпадение по `user_id`. Hash Join — хорош для неотсортированных и неиндексированных соединений.

### Пример 4: `JOIN` с Nested Loop (при наличии индекса)

```
CREATE INDEX idx_orders_user_id ON orders(user_id);
ANALYZE orders;

EXPLAIN SELECT * FROM users JOIN orders ON users.id = orders.user_id;
```

```
Nested Loop  (cost=0.25..18.70 rows=10 width=64)
  ->  Seq Scan on users  (cost=0.00..1.05 rows=5 width=32)
  ->  Index Scan using idx_orders_user_id on orders  (cost=0.25..3.45 rows=2 width=32)
        Index Cond: (user_id = users.id)
```

**Комментарий:** PostgreSQL перебирает `users`, и по каждому ищет заказы через индекс. Это эффективно при малом числе строк слева.

---

### Пример 5: Устаревшая статистика

```
-- удалим индекс и испортим статистику:
DROP INDEX idx_orders_user_id;
UPDATE orders SET user_id = 99999;
-- но не запускаем ANALYZE

EXPLAIN ANALYZE SELECT * FROM users JOIN orders ON users.id = orders.user_id;
```

```
Hash Join  (cost=1.05..23.30 rows=10 width=64) (actual time=0.050..0.150 rows=0 loops=1)
  Hash Cond: (orders.user_id = users.id)
  ->  Seq Scan on orders  (cost=0.00..11.40 rows=140 width=32) (actual time=0.030..0.050 rows=0 loops=1)
  ->  Hash  (cost=1.00..1.00 rows=5 width=32) (actual time=0.010..0.010 rows=5 loops=1)
        ->  Seq Scan on users  (cost=0.00..1.00 rows=5 width=32) (actual time=0.005..0.006 rows=5 loops=1)
Planning Time: 0.120 ms
Execution Time: 0.190 ms
```

**Комментарий:** планировщик думает, что будет найдено 10 строк, но `rows=0` показывает реальность. Надо выполнить:

```
ANALYZE orders;
```

---

> [!tip] Рекомендации
> 
> - Добавляй `EXPLAIN (ANALYZE, BUFFERS)` для полной картины.
>     
> - Обращай внимание на `rows` vs `actual rows` — если сильно расходятся, статистика устарела.
>     
> - Изучай `loops`: если внезапно стало 1000+ — возможно, ошибка в `JOIN`-стратегии.
>     
> - Используй `ANALYZE` после больших изменений в таблицах.
>     
> - Используй `pg_stat_statements` и `auto_explain` в проде.
>     

---

> [!warning] Подводные камни
> 
> - `EXPLAIN` не выполняет запрос, но `EXPLAIN ANALYZE` — **выполняет**!
>     
> - `cost` — не время, а относительная оценка. Сравнивай внутри одного плана.
>     
> - Индекс не всегда лучше — при малом селекте `Seq Scan` может быть быстрее.
>     
> - План может меняться от запуска к запуску: PostgreSQL адаптивен.
>     

---

> [!faq] Часто задаваемые вопросы
> 
> **Q:** Почему PostgreSQL не использует мой индекс?  
> **A:** Возможно, выборка слишком большая, или статистика устарела.
> 
> **Q:** Стоит ли всегда добавлять `ANALYZE` после `VACUUM`?  
> **A:** Да. `VACUUM` очищает, а `ANALYZE` обновляет статистику.
> 
> **Q:** Как ускорить сложный `JOIN`?  
> **A:** Добавить индекс по join-полю, уменьшить объём промежуточных данных, использовать CTE.
> 
> **Q:** Чем отличается Hash Join от Nested Loop?  
> **A:** Hash эффективен при больших объёмах и отсутствии индексов. Nested Loop — хорош при малом объёме слева и наличии индекса справа.

---

## Полезные ссылки

- https://www.postgresql.org/docs/current/sql-explain.html
    
- https://www.postgresql.org/docs/current/using-explain.html
