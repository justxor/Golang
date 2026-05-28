# 🟡 Slices и Maps (Q26–Q40)

> Любимая тема собесов. Большинство багов в Go-коде здесь.

---

## Q26
### Что такое slice внутри? Из чего состоит?

**Slice — header из 3 полей** (24 байта на 64-bit):

```go
type sliceHeader struct {
    Data unsafe.Pointer  // указатель на underlying array
    Len  int             // текущая длина
    Cap  int             // ёмкость
}
```

**Slice сам — это значение** (передаётся копией), но `Data` указывает на общий массив. Поэтому модификации элементов видны вызывающему, а `append` — нет.

```go
func modify(s []int)     { s[0] = 999 }   // ✅ видно снаружи
func extend(s []int)     { s = append(s, 1) }  // ❌ не видно
```

**Создание:**
```go
s1 := []int{1, 2, 3}              // literal
s2 := make([]int, 5)              // len=5, cap=5, нули
s3 := make([]int, 0, 10)          // len=0, cap=10
s4 := arr[2:5]                    // slice от массива
var s5 []int                      // nil slice
```

---

## Q27
### Чем отличается array от slice?

| | Array `[5]int` | Slice `[]int` |
|---|---------------|---------------|
| Размер | Часть типа, фиксирован | Динамический |
| Передача | По значению (полная копия!) | Header (24 байта) |
| Сравнение | `==` работает | Только с `nil` |
| Длина | `len(a)` (compile-time) | `len(s)`, `cap(s)` |

```go
var a [3]int = [3]int{1, 2, 3}
var b [4]int                   // ❌ a и b разные ТИПЫ

arr := [...]int{1, 2, 3, 4}    // тип [4]int — компилятор посчитал
```

**На практике array используются редко** (внутри slice, ключи map, фиксированные буферы).

---

## Q28
### Что такое len() и cap() у slice?

- **`len(s)`** — количество элементов, к которым можно обратиться `s[i]`.
- **`cap(s)`** — размер underlying array от начала slice до конца.

```go
arr := [10]int{0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
s := arr[2:5]   // элементы 2, 3, 4

len(s)  // 3
cap(s)  // 8 (от индекса 2 до конца массива = 10-2)
```

**Полный синтаксис slicing — 3 индекса:**
```go
s := arr[2:5:7]  // [start:end:max_cap]
len(s)  // 3 (5-2)
cap(s)  // 5 (7-2)
```

Полезно когда хочешь ограничить рост slice через `append` — следующий append выделит новый массив.

---

## Q29
### Как растёт slice при append?

Когда `len == cap`, `append` выделяет новый underlying array. **Стратегия (упрощённо, Go 1.18+):**
- Если новая длина ≤ 256 → удвоение: `newCap = 2 * oldCap`.
- Если > 256 → плавный рост: `newCap = oldCap + (oldCap + 3*256) / 4` (~25%).

```go
s := make([]int, 0)
for i := 0; i < 10; i++ {
    s = append(s, i)
    fmt.Printf("len=%d cap=%d\n", len(s), cap(s))
}
// len=1 cap=1
// len=2 cap=2
// len=3 cap=4
// len=4 cap=4
// len=5 cap=8
// ...
```

**Лайфхак:** если знаешь итоговый размер — `make([]T, 0, N)` избежит реаллокаций.

---

## Q30
### Подводные камни append: shared underlying array

**Классическая ловушка:**

```go
a := []int{1, 2, 3, 4, 5}
b := a[:3]               // [1 2 3], cap=5
c := append(b, 100)      // [1 2 3 100], НО переписал a[3]!

fmt.Println(a)  // [1 2 3 100 5]  ⚠️
fmt.Println(b)  // [1 2 3]
fmt.Println(c)  // [1 2 3 100]
```

**Почему:** `b` имеет `cap=5`, поэтому `append` записал в существующий массив, не аллоцируя новый.

**Решения:**
1. **Full slice expression** для изоляции:
```go
b := a[:3:3]  // cap = 3, append вынужден аллоцировать
c := append(b, 100)
// a = [1 2 3 4 5] — не изменено
```

2. **Explicit copy:**
```go
b := make([]int, len(a[:3]))
copy(b, a[:3])
```

3. **`slices.Clone` (Go 1.21+):**
```go
import "slices"
b := slices.Clone(a[:3])
```

---

## Q31
### Как правильно копировать slice?

```go
src := []int{1, 2, 3, 4, 5}

// 1. make + copy
dst := make([]int, len(src))
copy(dst, src)

// 2. append (идиома)
dst := append([]int(nil), src...)

// 3. slices.Clone (Go 1.21+)
dst := slices.Clone(src)
```

**❌ Анти-паттерн:** `dst := src` — это копия header, underlying array общий.

**Замер:** все три способа одинаковы по производительности (~250 ns/op для 10 элементов).

---

## Q32
### Как удалить элемент из slice?

**По индексу (порядок важен):**
```go
s := []int{1, 2, 3, 4, 5}
i := 2
s = append(s[:i], s[i+1:]...)  // [1 2 4 5]
// или (Go 1.21+):
s = slices.Delete(s, i, i+1)
```

**По индексу (порядок не важен — O(1)):**
```go
s[i] = s[len(s)-1]
s = s[:len(s)-1]
```

**По значению (с фильтром, in-place):**
```go
n := 0
for _, v := range s {
    if v != target {
        s[n] = v
        n++
    }
}
s = s[:n]
// или: slices.DeleteFunc(s, func(v int) bool { return v == target })
```

**Подводный камень:** после `append(s[:i], s[i+1:]...)` хвост массива остаётся в памяти (GC не освободит), если slice держит указатели — это утечка.

Решение:
```go
// Обнули конец
for k := n; k < len(s); k++ {
    s[k] = 0  // или nil для указателей
}
```

---

## Q33
### Что такое nil slice vs empty slice?

```go
var a []int                // nil slice: header={nil, 0, 0}
b := []int{}               // empty slice: header={ptr, 0, 0}
c := make([]int, 0)        // empty slice

a == nil  // true
b == nil  // false
len(a) == len(b)  // true (оба 0)
```

**Для большинства операций одинаковы:**
- `len`, `cap`, `for range` работают с обоими.
- `append(nil, ...)` работает.

**Различия:**
- JSON: `nil` → `null`, `[]int{}` → `[]`.
- Сравнение с `nil`.

**Идиома:** возвращай `nil`, а не `[]int{}`, кроме случаев JSON API.

---

## Q34
### Что такое map внутри? Hash table устройство

**Map** реализован как **hash table** (хеш-таблица) с раздельной цепочкой через **бакеты**.

**Упрощённая модель:**
```go
type hmap struct {
    count     int
    flags     uint8
    B         uint8           // log2(число бакетов)
    buckets   unsafe.Pointer  // массив бакетов
    oldbuckets unsafe.Pointer // при росте
    // ...
}

type bmap struct {
    tophash [8]uint8  // верхние биты хеша
    keys    [8]K
    values  [8]V
    overflow *bmap   // если коллизия > 8
}
```

**Алгоритм lookup:**
1. Хеш ключа: `h = hash(key)`.
2. Индекс бакета: `h & (2^B - 1)`.
3. Сравнить `tophash` с верхними битами `h` (быстрый отсев).
4. При совпадении — сравнить ключ.
5. Если бакет переполнен — переход по overflow.

**Рост (rehashing):** при load factor > 6.5 — удвоение `B`. Происходит **постепенно** (incremental) во время операций.

---

## Q35
### Почему map не потокобезопасна?

Чтение и запись map из разных горутин **без синхронизации** — гонка данных. Runtime детектит и **сразу падает с panic**:

```
fatal error: concurrent map read and map write
fatal error: concurrent map writes
```

**Почему не сделали thread-safe by default?**
- Производительность: блокировки замедлят single-thread сценарии.
- Гранулярность блокировок — у разных задач разные требования.

**Race detector:**
```bash
go run -race main.go
go test -race ./...
```

---

## Q36
### Как сделать concurrent-safe map?

**1. `sync.Mutex` + map:**
```go
type SafeMap struct {
    mu sync.RWMutex
    m  map[string]int
}

func (s *SafeMap) Get(k string) (int, bool) {
    s.mu.RLock()
    defer s.mu.RUnlock()
    v, ok := s.m[k]
    return v, ok
}

func (s *SafeMap) Set(k string, v int) {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.m[k] = v
}
```

**2. `sync.Map`** — оптимизирована для двух паттернов:
- Каждый ключ записывается один раз, потом много читается.
- Несколько горутин работают с непересекающимися ключами.

```go
var sm sync.Map
sm.Store("key", 42)
v, ok := sm.Load("key")
sm.Delete("key")
sm.Range(func(k, v any) bool { return true })
```

**Когда `sync.Map` НЕ подходит:**
- Частые записи в одни и те же ключи — медленнее, чем `Mutex` + map.
- Нужны типы — `sync.Map` работает с `any`.

**3. Sharded map** (для high-throughput): разбить на N бакетов, каждый со своим mutex.

---

## Q37
### Почему порядок обхода map случайный?

**Намеренно рандомизирован** в Go 1.0, чтобы программисты не полагались на порядок (как баг в недетерминизме). Стартовая позиция итерации = случайный бакет + случайный offset.

```go
m := map[string]int{"a": 1, "b": 2, "c": 3}
for k, v := range m {  // порядок РАЗНЫЙ при каждом запуске
    fmt.Println(k, v)
}
```

**Как получить упорядоченный обход:**
```go
keys := make([]string, 0, len(m))
for k := range m {
    keys = append(keys, k)
}
sort.Strings(keys)
for _, k := range keys {
    fmt.Println(k, m[k])
}
// Go 1.23: maps.Keys() + slices.Sorted()
```

---

## Q38
### Как удалить элемент из map?

```go
delete(m, key)  // безопасно даже если key отсутствует
```

**Безопасно во время итерации:**
```go
for k, v := range m {
    if shouldDelete(v) {
        delete(m, k)  // ✅ разрешено
    }
}
```

**Память:** удаление не уменьшает `cap` (бакеты не освобождаются). Если нужно — `m = nil` или новая map.

**Clear (Go 1.21+):**
```go
clear(m)  // удаляет все элементы
```

---

## Q39
### Что вернёт m[key] если ключа нет?

Zero value соответствующего типа значения:

```go
m := map[string]int{}
v := m["missing"]  // 0

mm := map[string]*User{}
u := mm["missing"]  // nil
```

**Это удобно**, но создаёт ловушку — нельзя отличить «нет ключа» от «значение 0».

---

## Q40
### Comma-ok идиома для map

```go
v, ok := m[key]
if !ok {
    // ключа нет
}
```

**`ok`** — bool, `true` если ключ существует.

**Использования:**
```go
// 1. Проверка существования
if _, ok := m[k]; ok { ... }

// 2. Default value
v, ok := m[k]
if !ok { v = defaultValue }

// 3. Set с условием
if _, exists := m[k]; !exists {
    m[k] = newValue
}
```

**В set-семантике (map как множество):**
```go
set := map[string]struct{}{}  // 0 байт на значение
set["a"] = struct{}{}
_, exists := set["a"]
```

---

⬅️ [Типы и память](02-types-memory.md) · ➡️ [Concurrency](04-concurrency.md)
