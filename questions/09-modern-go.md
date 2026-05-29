# 🆕 Новые версии Go: 1.21 → 1.24

> Вопросы по фичам последних релизов. На собеседовании часто проверяют, следите ли вы за языком.

⬅️ [08-production.md](08-production.md) · [README](../README.md)

---

## Q101
### Что нового в Go 1.21? Главные фичи

**TL;DR:** Пакеты `log/slog`, `slices`, `maps`; встроенные `min`, `max`, `clear`; `sync.OnceFunc/OnceValue/OnceValues`; `context.WithoutCancel`, `AfterFunc`; PGO стало production-ready; `GOMEMLIMIT` + улучшения GC.

**Разбор по группам:**

**Stdlib:**
- `log/slog` — структурированное логирование (level, attrs, handlers, JSON/Text).
- `slices` — generic-операции над слайсами: Sort, BinarySearch, Clone, Equal, Contains, Index, Delete, Insert.
- `maps` — Keys, Values, Clone, Equal, Copy, DeleteFunc.
- `cmp` — `cmp.Compare`, `cmp.Or` для генериков.

**Builtins:**
```go
min(1, 2, 3)        // 1 — variadic, любое Ordered
max(3.14, 2.71)     // 3.14
clear(m)            // удалить все ключи из map
clear(s)            // обнулить элементы slice (важно для GC)
```

**Concurrency:**
```go
var once = sync.OnceValue(func() *Config {
    return loadConfig()
})
cfg := once() // лениво, один раз, потокобезопасно
```

**Pitfalls:**
- `clear(s)` обнуляет ЭЛЕМЕНТЫ, но не меняет len. Для урезания: `s = s[:0]`.
- `slices.Sort` изменяет исходный слайс, `slices.Sorted` возвращает новый.

---

## Q102
### Что такое log/slog и зачем он нужен?

**TL;DR:** Стандартный structured logger из Go 1.21. Уровни (Debug/Info/Warn/Error), атрибуты (`slog.String`, `slog.Int`), handlers (TextHandler, JSONHandler), groups, дочерние логгеры через `With`. Заменяет `log` для прод-систем.

**Пример:**
```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
    Level: slog.LevelDebug,
    AddSource: true,
}))
slog.SetDefault(logger)

slog.Info("user login",
    slog.String("user_id", "42"),
    slog.Duration("took", 120*time.Millisecond),
)
// {"time":"...","level":"INFO","msg":"user login","user_id":"42","took":120000000}
```

**С контекстом:**
```go
// LogAttrs быстрее (без аллокаций на boxing)
slog.LogAttrs(ctx, slog.LevelInfo, "request done",
    slog.String("method", "GET"),
    slog.Int("status", 200),
)

// child logger с общими полями
reqLog := logger.With("request_id", reqID)
reqLog.Info("processing")
```

**Преимущества над zap/zerolog:**
- Часть стандартной библиотеки — нет deps.
- Handler API позволяет переключать форматы без изменения логирующего кода.
- Замедление между slog и zap минимально для типового кода.

**Pitfalls:** обычный `slog.Info("msg", "k", v)` медленнее, чем `LogAttrs` с типизированными атрибутами.

---

## Q103
### Что такое slices и maps пакеты?

**TL;DR:** Generic-утилиты для слайсов и мап. Заменяют ручные циклы и копипасту, дают идиоматичные read-only/mutating операции. Появились в Go 1.21.

**slices — самое нужное:**
```go
slices.Contains(s, x)         // bool
slices.Index(s, x)            // int, -1 если нет
slices.Sort(s)                // sort in-place, Ordered
slices.SortFunc(s, cmp)       // custom comparator
slices.Reverse(s)             // in-place
slices.Clone(s)               // полная копия
slices.Equal(a, b)            // deep equality (== по элементам)
slices.Delete(s, i, j)        // удалить s[i:j], вернуть новый slice
slices.Insert(s, i, vs...)    // вставить
slices.BinarySearch(s, x)     // (idx, found) — для отсортированных
slices.Max(s), slices.Min(s)  // panic на пустом
```

**maps:**
```go
maps.Keys(m)     // iter.Seq[K] в 1.23, []K в 1.21
maps.Values(m)   // аналогично
maps.Clone(m)    // shallow copy
maps.Equal(a, b) // bool
maps.Copy(dst, src)
maps.DeleteFunc(m, func(k, v) bool)
```

**Важно про версии:**
- В 1.21 `maps.Keys` возвращал `[]K`.
- В 1.23 переписали на `iter.Seq[K]` — нужно `slices.Collect(maps.Keys(m))`, чтобы получить слайс.

---

## Q104
### Что такое sync.OnceFunc / OnceValue / OnceValues?

**TL;DR:** Удобные обёртки над `sync.Once`, возвращающие функцию, которую можно дёргать многократно — выполнится тело ровно один раз. Появились в Go 1.21.

**Пример:**
```go
// до Go 1.21:
var (
    once sync.Once
    cfg  *Config
)
func getCfg() *Config {
    once.Do(func() { cfg = loadConfig() })
    return cfg
}

// с Go 1.21:
var getCfg = sync.OnceValue(loadConfig)
// getCfg() лениво один раз вычисляет результат

// OnceValues — для (T, error)
var getDB = sync.OnceValues(func() (*sql.DB, error) {
    return sql.Open("postgres", dsn)
})

// OnceFunc — без возврата
var initLogger = sync.OnceFunc(func() {
    slog.SetDefault(buildLogger())
})
```

**Pitfalls:**
- Если функция паникует — `OnceValue` закеширует panic и будет повторно паниковать при каждом вызове.
- Эти функции потокобезопасны — можно использовать как глобалы.

---

## Q105
### Что нового в Go 1.22?

**TL;DR:** Главное — **исправление loop variable capture** (каждая итерация — новая переменная), числовой `range int`, улучшенный HTTP-роутер с паттернами и методами, `math/rand/v2`, `slices.Concat`.

**1. Loop variable scoping (game changer):**
```go
// До Go 1.22 — классический баг:
for _, v := range items {
    go func() { fmt.Println(v) }() // все печатают последний v!
}

// С Go 1.22 — v НОВАЯ на каждой итерации:
for _, v := range items {
    go func() { fmt.Println(v) }() // OK, печатает разные значения
}
```

**2. Range over integer:**
```go
for i := range 10 {
    fmt.Println(i) // 0..9
}
// заменяет for i := 0; i < 10; i++ { ... }
```

**3. HTTP ServeMux с паттернами и методами:**
```go
mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", getUser)
mux.HandleFunc("POST /users", createUser)
mux.HandleFunc("DELETE /users/{id}", deleteUser)

func getUser(w http.ResponseWriter, r *http.Request) {
    id := r.PathValue("id")
    // ...
}
```
Раньше для этого нужен был chi/gorilla/gin.

**4. math/rand/v2:**
- Новый API на дженериках, лучший RNG (ChaCha8) по умолчанию, без глобального лока.

**Pitfalls:**
- Изменение semantics цикла — БРЕКНГ-ЧЕНДЖ. Контролируется через `go` директиву в `go.mod`.
- Старый код, опиравшийся на shared loop var, может тихо сломаться (редко).

---

## Q106
### Что такое router в net/http Go 1.22? Зачем нужен сторонний?

**TL;DR:** Встроенный `http.ServeMux` получил паттерны вида `METHOD /path/{var}` и приоритет специфичных маршрутов. Покрывает 80% задач без gorilla/chi.

**Что умеет:**
- HTTP-методы в паттерне: `"GET /api/users/{id}"`.
- Path-параметры: `r.PathValue("id")`.
- Wildcard: `"GET /files/{path...}"` — захват хвоста пути.
- Host-matching: `"api.example.com/users"`.
- Приоритет: более конкретный паттерн выигрывает.

**Чего НЕ умеет (тут нужен chi/echo/gin):**
- Middleware-цепочки из коробки (только ручные обёртки).
- Regex в path-параметрах.
- Group-routing (хотя легко эмулируется prefix-сabmux-ом).
- Авто-валидация JSON, биндинг, error helpers.

**Пример с middleware:**
```go
func logMW(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        slog.Info("req", "path", r.URL.Path, "took", time.Since(start))
    })
}

mux := http.NewServeMux()
mux.HandleFunc("GET /users/{id}", getUser)
http.ListenAndServe(":8080", logMW(mux))
```

---

## Q107
### Что такое range-over-func в Go 1.23?

**TL;DR:** Можно делать `for x := range fn`, где `fn` — функция-итератор соответствующей сигнатуры. Унифицирует пользовательские итераторы. Тип `iter.Seq[T]`, `iter.Seq2[K,V]`.

**Сигнатуры итераторов:**
```go
type Seq[V]    = func(yield func(V) bool)
type Seq2[K,V] = func(yield func(K, V) bool)
```

**Пример: свой итератор для линейного списка:**
```go
func (l *List[T]) All() iter.Seq[T] {
    return func(yield func(T) bool) {
        for n := l.head; n != nil; n = n.next {
            if !yield(n.value) {
                return // потребитель сделал break
            }
        }
    }
}

for v := range list.All() {
    fmt.Println(v)
}
```

**Stdlib возвращает Seq:**
```go
for k, v := range maps.All(m) {  // Seq2
    fmt.Println(k, v)
}
for v := range slices.Values(s) { // Seq
    fmt.Println(v)
}

// собрать обратно в slice:
keys := slices.Collect(maps.Keys(m))
sorted := slices.Sorted(maps.Keys(m))
```

**Pitfalls:**
- `yield` возвращает `bool`: если `false` — потребитель прервал, итератор должен корректно завершиться (закрыть ресурсы).
- Через `defer` внутри итератора освобождайте файлы/коннекшены.

---

## Q108
### Что такое unique пакет и weak.Pointer в Go 1.23?

**TL;DR:** `unique` — интернирование (canonicalization) сравнимых значений: одинаковые строки/структуры хранятся в одном экземпляре. `weak.Pointer` (1.24) — слабая ссылка, не удерживает объект от GC.

**unique.Make:**
```go
import "unique"

h1 := unique.Make("hello")
h2 := unique.Make("hello")
// h1 == h2 (одна и та же Handle)
s := h1.Value() // "hello"
```

Полезно когда:
- Миллионы повторяющихся строк/ключей (метки метрик, имена полей).
- Хотите сравнивать за O(1) по Handle вместо string-cmp.

**weak.Pointer (Go 1.24):**
```go
import "weak"

obj := &BigCache{...}
wp := weak.Make(obj)

// потом:
if p := wp.Value(); p != nil {
    use(p)
} else {
    // GC уже собрал
}
```
Использование: кеши, observable-паттерн без leak-ов.

---

## Q109
### Что такое Generic Type Aliases (Go 1.24)?

**TL;DR:** Aliases теперь могут быть параметризованы дженериками. Раньше дженерики работали только в `type X[T any] = struct{...}` определениях, но не в алиасах.

**До Go 1.24:**
```go
type StringMap = map[string]string  // OK
type Mapping[K comparable, V any] = map[K]V  // ERROR в <1.24
```

**Go 1.24:**
```go
type Set[T comparable] = map[T]struct{}
type Result[T any]     = struct {
    Value T
    Err   error
}

func ok[T any](v T) Result[T] {
    return Result[T]{Value: v}
}
```
Зачем: рефакторинг (переименование типов без поломки API), удобные shorthand для длинных типов в библиотеках.

---

## Q110
### Что такое Swiss Tables и зачем их добавили в Go 1.24?

**TL;DR:** Go 1.24 переписал реализацию `map` на Swiss Tables — алгоритм от Google. Меньше памяти, быстрее lookup (особенно cache-miss). Прозрачно для пользователя.

**Что изменилось:**
- Старая реализация: bucket-ы по 8 элементов, открытая адресация.
- Новая: control bytes + groups, SIMD-friendly probing.
- Бенчмарки: lookups быстрее на 5–60%, memory footprint меньше для частично заполненных map.

**API не изменился** — никаких миграций. Просто пересобрать на Go 1.24.

**Что осталось как было:**
- Concurrent writes по-прежнему `fatal error`.
- Random iteration order сохранён.
- `sync.Map` отдельно (не на Swiss).

---

## Q111
### Что такое testing/synctest (экспериментально в Go 1.24)?

**TL;DR:** Пакет для детерминированного тестирования конкурентного кода: виртуальное время, контролируемая блокировка горутин. Решает проблему flaky tests с `time.Sleep`.

**Идея:**
```go
import "testing/synctest"

func TestTimeout(t *testing.T) {
    synctest.Run(func() {
        ctx, cancel := context.WithTimeout(context.Background(), time.Hour)
        defer cancel()

        // время виртуальное — час "проходит" мгновенно
        <-ctx.Done()
        if ctx.Err() != context.DeadlineExceeded {
            t.Fatal("expected deadline")
        }
    })
}
```

**Зачем:**
- Тесты на cancel/timeout без реальных задержек.
- Воспроизводимые race-сценарии.
- Заменяет нужду в моках clock-а.

**Статус:** Включается через `GOEXPERIMENT=synctest`.

---

## Q112
### Какие новые linter-проверки появились в go vet (1.22-1.24)?

**TL;DR:** В 1.22 добавили `loopclosure` предупреждение (теперь почти не нужно — фикс на уровне языка). В 1.23 — `stdversion` (использование фич, недоступных в `go` директиве). Усилены `printf`, `copylocks`.

**Примеры:**
```go
// stdversion ловит:
// go.mod: go 1.20
// в коде: clear(m)  // ERROR: clear requires 1.21+

// copylocks ловит копирование sync.Mutex:
func bad(m sync.Mutex) {} // vet: m passes Mutex by value

// printf ловит:
fmt.Printf("%d", "hello") // vet: wrong type
```

**Рекомендация:** запускать `go vet ./...` в CI + `golangci-lint` с включёнными `staticcheck`, `govet`, `errcheck`.

---

## Q113
### Что такое GOTOOLCHAIN и как работает с Go 1.21+?

**TL;DR:** Go 1.21+ умеет автоматически подкачивать нужную версию toolchain по `go.mod`. Переменная `GOTOOLCHAIN` управляет поведением.

**В go.mod:**
```
go 1.22
toolchain go1.23.4
```

Если установлен старый Go (например 1.21), но в `go.mod` указан `toolchain go1.23.4` — Go сам скачает 1.23.4 и использует его.

**Значения GOTOOLCHAIN:**
- `auto` (default) — использовать toolchain из go.mod при необходимости.
- `local` — только установленная версия (без авто-загрузки).
- `go1.22.5` — принудительно эта версия.
- `go1.22.5+auto` — минимум 1.22.5, иначе auto.

**CI/Docker:** ставьте `GOTOOLCHAIN=local`, если хотите контролировать версию через образ.

---

## Q114
### Что такое iter.Pull / iter.Pull2?

**TL;DR:** Превращает push-итератор (`iter.Seq`) в pull-итератор — функцию `next() (T, bool)`. Нужно, когда вам нужен ручной контроль над продвижением (мерж двух потоков, lookahead).

**Пример: merge двух отсортированных Seq:**
```go
func Merge[T cmp.Ordered](a, b iter.Seq[T]) iter.Seq[T] {
    return func(yield func(T) bool) {
        nextA, stopA := iter.Pull(a)
        defer stopA()
        nextB, stopB := iter.Pull(b)
        defer stopB()

        va, okA := nextA()
        vb, okB := nextB()
        for okA && okB {
            if va <= vb {
                if !yield(va) { return }
                va, okA = nextA()
            } else {
                if !yield(vb) { return }
                vb, okB = nextB()
            }
        }
        // хвост
        for okA { if !yield(va) { return }; va, okA = nextA() }
        for okB { if !yield(vb) { return }; vb, okB = nextB() }
    }
}
```

**Важно:** `stop()` нужно вызывать обязательно — иначе горутина итератора подвиснет.

---

## Q115
### Что нового в context (Go 1.21+)?

**TL;DR:** `context.WithoutCancel` — наследует значения, но не отмену; `context.AfterFunc` — регистрирует функцию, которая выполнится при cancel; `WithDeadlineCause`, `WithTimeoutCause` — кастомные причины отмены.

**WithoutCancel:**
```go
// в HTTP handler нужно записать аудит, даже если клиент отменил:
func handle(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    go writeAudit(context.WithoutCancel(ctx), data)
    // ctx внутри сохраняет values (trace_id и т.д.), но не отменится при отмене r
}
```

**AfterFunc:**
```go
stop := context.AfterFunc(ctx, func() {
    log.Println("cleanup on cancel")
})
defer stop() // отменить регистрацию, если не нужно
```

**Cause:**
```go
ctx, cancel := context.WithCancelCause(parent)
cancel(errors.New("user logged out"))

<-ctx.Done()
fmt.Println(ctx.Err())        // context.Canceled
fmt.Println(context.Cause(ctx)) // user logged out
```
Зачем: отличать причины отмены в логах/метриках.

---

## Q116
### Что такое min, max, clear builtins (Go 1.21)?

**TL;DR:** Встроенные generic-функции без импорта. `min/max` — variadic для любого `Ordered`. `clear` — обнуляет map/slice.

**Примеры:**
```go
a := min(3, 1, 4, 1, 5)  // 1
b := max(2.5, 7.1, 0.3)  // 7.1
s := min("abc", "abd")   // "abc" (лексикографически)

m := map[string]int{"a":1, "b":2}
clear(m) // m теперь len(m)==0

sl := []int{1, 2, 3}
clear(sl) // [0, 0, 0] — обнулил элементы, len остался 3
```

**Зачем clear(slice):**
- Если в слайсе указатели — обнуление помогает GC освободить ссылки.
```go
cache = append(cache[:0])
// ❌ cap остался, объекты по индексам >0 удерживаются GC

clear(cache)
cache = cache[:0]
// ✅ объекты освобождаются
```

**Pitfalls:**
- `min/max` panic на пустом variadic — но компилятор требует минимум 1 аргумент, так что practically безопасно.
- На NaN — поведение `min/max` такое же как у comparison: `NaN` "проигрывает".

---

## Q117
### Что такое cmp.Or и зачем нужен?

**TL;DR:** Go 1.22 добавил `cmp.Or` — возвращает первый ненулевой аргумент. Эквивалент null-coalescing-оператора (??) из других языков.

**Пример:**
```go
import "cmp"

name := cmp.Or(user.Nickname, user.Name, "anonymous")
// если Nickname != "" — он, иначе Name, иначе "anonymous"

port := cmp.Or(os.Getenv("PORT"), "8080")

// до 1.22 нужно было:
// name := user.Nickname
// if name == "" { name = user.Name }
// if name == "" { name = "anonymous" }
```

**Работает с любым comparable, проверяет на zero value.**

---

## Q118
### Изменения в encoding/json в новых версиях?

**TL;DR:** Идёт работа над `encoding/json/v2` (экспериментальный). В стандартном: фиксы корректности (1.21+), быстрее unmarshal в некоторых случаях. Опции `omitempty` пересматриваются — будет `omitzero` в 1.24.

**omitzero (Go 1.24):**
```go
type User struct {
    Name  string    `json:"name,omitempty"`  // пропускает ""
    Age   int       `json:"age,omitempty"`   // пропускает 0 — БАГ (0 валидный!)
    Email string    `json:"email,omitzero"`  // пропускает "" (1.24+)
    Born  time.Time `json:"born,omitzero"`   // пропускает zero Time
}
```
Разница: `omitempty` использует "is-empty" check (len==0, ==0, false), `omitzero` — `IsZero()` метод или сравнение с zero value.

**json/v2 (experimental, ~1.24-1.25):**
- Быстрее, корректнее (правильный handling Unmarshaler для указателей).
- Новые опции: `json:",case:ignore"`, `json:",inline"`.
- Streaming API: `jsontext`.
- Включается через `GOEXPERIMENT=jsonv2`.

---

## Q119
### Что такое math/rand/v2 и зачем v2?

**TL;DR:** Go 1.22 ввёл `math/rand/v2` — новый API без глобального лока, на дженериках, с лучшим генератором (ChaCha8). Старый `math/rand` остался для обратной совместимости.

**Сравнение:**
```go
// math/rand (старый):
rand.Intn(100)            // глобальный sync.Mutex
r := rand.New(rand.NewSource(42)) // mutex per source

// math/rand/v2:
import "math/rand/v2"
rand.IntN(100)            // быстрее, без глобального лока
rand.Int32(), rand.Uint64()
rand.N[int64](100)        // generic, по типу T

r := rand.New(rand.NewChaCha8([32]byte{}))
```

**Что изменилось:**
- Глобальный источник стал per-goroutine (через `runtime`).
- Default RNG — криптографически "приличный" (но не для криптографии всё равно — используйте `crypto/rand`).
- API без `Source` интерфейса (он усложнял реализацию).

---

## Q120
### Какие best practices при миграции на новую версию Go?

**TL;DR:** Поднимать `go` директиву в `go.mod` постепенно. Прогнать `go test -race ./...`, `go vet ./...`. Особо тестировать места с горутинами + range (Go 1.22 loop var fix).

**Чек-лист:**
1. Обновить toolchain: `go install golang.org/dl/go1.23@latest && go1.23 download`.
2. Поднять `go 1.22` в `go.mod` (включает loop var fix).
3. Прогнать `go test -race -count=10 ./...` — поймать оставшиеся races.
4. Запустить `gopls` / `staticcheck` — обновить deprecated API:
   - `ioutil.ReadAll` → `io.ReadAll`.
   - `interface{}` → `any` (опционально).
   - Ручные `sync.Once` wrappers → `sync.OnceValue`.
5. Включить `PGO` для hot-path сервисов: `go build -pgo=auto`.
6. Прописать `GOMEMLIMIT` в контейнерах.
7. Перейти на `log/slog` для новых сервисов.

**Что НЕ ломается:**
- Подавляющее большинство кода работает as-is.
- Go бережёт обратную совместимость (Go 1 compatibility promise).

**Что может сломаться:**
- Зависимость на старый `math/rand` detarminism (seed=1 даёт другую последовательность? — нет, гарантия осталась).
- Generic type aliases в библиотеках, требующих 1.24.
- Поведение range в редких случаях, где код опирался на shared loop var (обычно баг и так был).

---

## 📊 Сводная таблица: новые фичи по версиям

| Версия | Главное |
|--------|---------|
| **1.21** | `log/slog`, `slices`, `maps`, `cmp`, `min/max/clear`, `sync.OnceFunc/Value/Values`, `context.WithoutCancel/AfterFunc`, PGO production, `GOMEMLIMIT` |
| **1.22** | **Loop var per-iteration**, `range int`, HTTP-роутер с методами/паттернами, `math/rand/v2`, `slices.Concat`, `cmp.Or` |
| **1.23** | **range-over-func** (`iter.Seq/Seq2`), `iter.Pull`, `unique`, generic `maps.Keys/Values` как `Seq`, timer Reset/Stop изменения |
| **1.24** | **Generic type aliases**, **Swiss tables maps** (быстрее), `weak.Pointer`, `testing/synctest` (experimental), `omitzero` в json, `crypto/mlkem` (post-quantum) |

---

## 🔗 Полезное

- [Release notes Go 1.21](https://go.dev/doc/go1.21)
- [Release notes Go 1.22](https://go.dev/doc/go1.22)
- [Release notes Go 1.23](https://go.dev/doc/go1.23)
- [Release notes Go 1.24](https://go.dev/doc/go1.24)
- TG: [@Golang_google](https://t.me/Golang_google), [@ai_machinelearning_big_data](https://t.me/ai_machinelearning_big_data)

---

⬅️ [08-production.md](08-production.md) · [README](../README.md)
