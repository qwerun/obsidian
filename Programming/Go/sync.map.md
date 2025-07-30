## Введение

`sync.Map` решает главную боль конкурентных Go‑приложений — **безопасный и быстрый доступ к общей карте** без явных мьютексов. После улучшений в Go 1.22‑1.24 (новый **HashTrieMap** и тонкая GC‑интеграция) структура стала реальной альтернативой паре `map` + `sync.RWMutex`.

> [!info] Цель: быстро показать, **когда** использовать `sync.Map`, **как** она устроена и **чем** отличается от `map`+мьютекс.

---

## Внутреннее устройство `sync.Map`

|Компонент|Назначение|Особенности|
|---|---|---|
|`readOnly`|Lock‑free чтение|Хранится в `atomic.Value`|
|`dirty`|Записи/новые ключи|Под мьютексом, продвигается в `readOnly` по счётчику промахов|
|HashTrieMap (1.24)|Хэш‑trie|O(log N) lookup, лучшая cache‑locality|

**Алгоритм:** чтение → fast‑path в `readOnly`; промах → `dirty` (slow‑path) → по условию «misses ≥ len(dirty)» карты меняются. GC видит старую карту как единый объект и забирает её в фоне.

_Выжимка_

- Чтения lock‑free, записи блокируют только `dirty`.
    
- Под нагрузкой read‑heavy contention ≈ 0.
    
- HashTrieMap стабилизирует латентность при > 10M ключей.
    

---

## `map` + `sync.(RW)Mutex`

|   |   |   |
|---|---|---|
|Путь|Действия|Латентность|
|Fast|`RLock→lookup→RUnlock`|низкая при few writes|
|Slow|`Lock→write→Unlock`|блокирует все чтения|

- При write‑heavy доля slow‑path растёт → contention.
    
- Resize мапы удваивает память, создаёт GC‑пик.
    

---

## Сценарии и выбор

|   |   |
|---|---|
|Условие|Лучше использовать|
|Read > 90 %, высокая кардинальность|`**sync.Map**`|
|Write ≥ 25 %, hot‑key|`map` + sharded `RWMutex`|
|Нужен снапшот без копирования|`sync.Map.Range`|
|Память критична, ключей < 1 k|Обычный `map` + мьютекс|

> [!tip] Замерьте **write‑ratio** и **p95 latency** — это главный критерий.

---

## Бенчмарки (Go 1.24, M1 Max, 10 CPU)

```
BenchmarkSyncMap_Read95W5   4.5 ns/op   0 B/op   0 allocs/op
BenchmarkMutexMap_Read95W5  9.1 ns/op   0 B/op   0 allocs/op
BenchmarkSyncMap_Write30    85 ns/op   16 B/op   1 alloc/op
BenchmarkMutexMap_Write30   52 ns/op    0 B/op   0 alloc/op
```

- `sync.Map` быстрее в read‑heavy, медленнее в write‑heavy.
    
- Аллокация на запись — внутренняя обёртка `*entry`.
    

---

## Чек‑лист выбора

1. **write ≤ 10 %?** — смело берите `sync.Map`.
    
2. **Hot‑key?** — шардируйте `map`, либо ждите HashTrieMap.
    
3. **Нужен lock‑free Range?** — `sync.Map`.
    
4. **Профиль write > 30 %** — `RWMutex` или channel‑шард.
    

---

## Edge‑cases

- Ключи без `==` (срезы) не подходят и там и там.
    
- `LoadOrStore` — используйте для lazy‑инициализации, но не кладите тяжёлый конструктор внутрь.
    
- Нельзя «очистить» `sync.Map` — только создать новую.
    

> [!warning] Не смешивайте своё шардирование с HashTrieMap — получите неоптимальный кэш.

---

## Примеры

### `sync.Map`

```
var m sync.Map
m.Store("id", 1)
if v, ok := m.Load("id"); ok { fmt.Println(v) }
m.Range(func(k, v any) bool { fmt.Println(k, v); return true })
```

### `map` + `RWMutex`

```
var (
  mu sync.RWMutex
  mm = make(map[string]int)
)
mu.Lock(); mm["id"] = 1; mu.Unlock()
mu.RLock(); _ = mm["id"]; mu.RUnlock()
```

---

> [!faq] 
> **Q:** Почему `sync.Map` аллоцирует на запись?  
> **A:** Создаётся `entry`, обёртка вокруг value.  
> 
> **Q:** Можно ли заменить Redis‑кэш на `sync.Map`?  
> **A:** Нет, она in‑process. Используйте для локальных кэшей.  
> 
> **Q:** Какой размер ключей критичен?  
> **A:** При > 1 M уникальных ключей HashTrieMap показывает стабильное O(log N).

---
## Заключение

`sync.Map` — оружие для **read‑heavy** кейсов, где важен lock‑free доступ и минимальные GC‑паузы. При write‑heavy нагрузке старый добрый `map` + `RWMutex` всё ещё быстрее и предсказуемее. **Измеряйте** на своей нагрузке, прежде чем выбрать.
