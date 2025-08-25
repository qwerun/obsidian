# `sync.Map` — конкурентная хэш‑таблица в Go

> [!abstract] **TL;DR** `sync.Map` — это оптимизированная под «много чтений / мало записей» конкурентная структура. Внутри две обычные `map[K]*entry`: _read‑only_ и _dirty_. Чтения из _read_ идут без блокировок, изменения попадают в _dirty_ под одним `sync.Mutex`. Когда счётчик промахов (`misses`) достигает размера _dirty_, она мгновенно «повышается» и становится новым _read_.

---

## 1  Когда выбирать `sync.Map`

> [!tip] Бери `sync.Map`, если…
> 
> - ключей много, заранее не известны;
>     
> - профиль — ≥ 80 % чтений;
>     
> - нужна ленивость создания значений (`LoadOrStore`).
>     

В остальных сценариях **map** **+** **sync.RWMutex** часто быстрее и типобезопаснее.

## 2  API‑шпаргалка

```
var m sync.Map
m.Store(k, v)              // запись
v, ok := m.Load(k)         // чтение
actual, loaded := m.LoadOrStore(k, v0)
old, swapped := m.CompareAndSwap(k, oldV, newV)
m.Delete(k)                // логическое удаление
m.Range(func(k, v any) bool { … return true })
```

Все операции, кроме `Range`, атомарны по отношению друг к другу.

## 3  Внутреннее устройство

```
Map
├─ mu    sync.Mutex           ← защищает dirty и служебные поля
├─ read  *readOnly (atomic)   ← «снимок», только читается
├─ dirty map[key]*entry       ← рабочая копия под mu
└─ misses int                 ← счётчик промахов

readOnly
├─ m       map[key]*entry     ← неизменяемая карта
└─ amended bool               ← есть ли ключи только в dirty

entry
└─ p atomic.Pointer[value]    ← *v | nil | expunged
```

### 3.1  Путь **чтения** (`Load`)

1. Атомарно берём указатель `read`.
    
2. Ищем ключ в `read.m` (обычный lookup Swiss‑таблицы).
    
3. Из найденного `*entry` читаем `entry.p`.
    
    - Если `p == nil` ➜ ключ удалён (slow‑path).
        
    - Если `p == expunged` ➜ ключ окончательно удалён (slow‑path).
        
    - Иначе возвращаем значение **без мьютекса**.
        
4. При промахе и `read.amended = true` идём в _dirty_ под `mu`, `misses++`.
    

### 3.2  Путь **записи** (`Store`) 

- Быстрый: ключ в `read`, `entry.p.CompareAndSwap`.
    
- Медленный: под `mu` создаём/обновляем запись в _dirty_. Если `dirty == nil`, копируем в неё **все** указатели из текущего снимка (O(n) поверхностная копия).
    

### 3.3  Promotion

Когда `misses ≥ len(dirty)`, выполняется

```
read = &readOnly{m: dirty}
dirty, misses = nil, 0
```

Старый снимок становится мусором для GC.

> [!faq] Часто задаваемое 
> **Q:** Сколько мьютексов в `sync.Map`?  
> **A:** Один `mu`. Чтения lock‑free.
> 
> **Q:** Можно ли атомарно обновить два ключа?  
> **A:** Нет, `sync.Map` не предоставляет групповых транзакций; берите `map`+`Mutex`.
> 
> **Q:** Когда появятся дженерики?  
> **A:** С Go 1.21 есть `type Map[K comparable, V any] struct …`.
> 
> **Q:** Почему значения хранятся через `*entry`?  
> **A:** Чтобы _read_ и _dirty_ делили одни и те же объекты и изменения были видны без копий.

## 4  Жизненный цикл ключа (high‑level)

1. **Store** нового ключа → попадает в _dirty_, `read.amended = true`.
    
2. **Load** пропускает в _read_, находит в _dirty_, `misses++`.
    
3. После N промахов → _dirty_ ➜ новый _read_, `dirty=nil`.
    
4. **Delete** ставит `entry.p = nil` (lock‑free) → позднее станет `expunged`.
    

## 5  Производительность (Go 1.24)

- Чтение из `sync.Map` ~= обычный `map` (Swiss Tables) — два атомарных load’а сверху.
    
- Запись медленней `map+Mutex`, т.к. копирование при создании _dirty_ и CAS.
    
- Память: +10‑20 % из‑за двух карт и sentinel‑значений.
    

Benchmarks от Go team показывают выигрыш, когда соотношение **reads/writes ≥ 4:1**.


---

### Источники

1. Victoriametrics — _Go sync.Map: The Right Tool for the Right Job_, Oct 04 2024 https://victoriametrics.com/blog/go-sync-map/
    
2. Dave Cheney — _How the Go runtime implements maps efficiently_, May 29 2018 https://dave.cheney.net/2018/05/29/how-the-go-runtime-implements-maps-efficiently-without-generics
    
3. Go 1.24 Release Notes — https://go.dev/doc/go1.24
    
4. Go blog — _Faster Go maps with Swiss Tables_, Feb 26 2025 https://go.dev/blog/swisstable
    
5. `src/runtime/map.go` (Go tip) https://go.googlesource.com/go/+/refs/heads/master/src/runtime/map.go
    
6. GitHub Issue #70683 — _sync: replace Map implementation with HashTrieMap_ https://github.com/golang/go/issues/70683
    
7. GitHub Issue #73015 — _Export HashTrieMap for external use_ https://github.com/golang/go/issues/73015
    
8. Go package docs — `sync` https://pkg.go.dev/sync

