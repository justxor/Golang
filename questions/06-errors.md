# 🟡 Ошибки и panic (Q71–Q80)

> Обработка ошибок в Go — не как в Java/Python. Подробный разбор подходов.

---

## Q71
### Как обрабатываются ошибки в Go?

Go отвергает исключения. Ошибки — это **значения** (`error` интерфейс), возвращаемые из функций.

```go
type error interface {
    Error() string
}
```

**Идиома:**
```go
f, err := os.Open("file.txt")
if err != nil {
    return fmt.Errorf("open file: %w", err)
}
defer f.Close()
```

**Плюсы:**
- Ошибки видны в сигнатуре.
- Нет невидимых потоков управления (try/except).
- Программист обязан явно обработать.

**Минусы:**
- Boilerplate: `if err != nil` много.
- Легко проигнорировать (`_`).
- Stack trace по умолчанию не сохраняется (нужно wrap).

**Создание ошибок:**
```go
errors.New("message")
fmt.Errorf("failed at %d: %w", i, err)
```

---

## Q72
### Чем отличается error от panic?

| | `error` | `panic` |
|---|---------|---------|
| Когда | Ожидаемые ошибки (файла нет, network) | Программные баги (nil deref, out of bounds) |
| Поток | Обычный return | Раскрутка стека до recover |
| Возврат | Явный | Через recover |
| Использование | 99% случаев | Редко |

**Правило:** ошибки бизнес-логики — `error`. Невозможные ситуации (баг программиста) — `panic`.

```go
// error — ожидаемо
if id < 0 { return fmt.Errorf("invalid id: %d", id) }

// panic — программный баг
if internalState == nil { panic("state must be initialized") }
```

**Panic тоже даёт:**
- Полный stack trace.
- Останавливает программу (или goroutine).
- Может быть перехвачена `recover()`.

---

## Q73
### errors.Is vs errors.As vs ==

**`errors.Is(err, target)`** — проверка, что в цепочке ошибок есть конкретное значение.

```go
_, err := os.Open("x")
if errors.Is(err, os.ErrNotExist) {
    // независимо от того, обёрнута ли
}
```

**`errors.As(err, &target)`** — извлечение конкретного типа ошибки из цепочки.

```go
var pathErr *os.PathError
if errors.As(err, &pathErr) {
    fmt.Println(pathErr.Path)
}
```

**`==`** — буквальное сравнение значений интерфейса. Работает только если ошибка не обёрнута.

```go
if err == io.EOF { ... }  // работает
wrapped := fmt.Errorf("context: %w", io.EOF)
wrapped == io.EOF       // false
errors.Is(wrapped, io.EOF)  // true
```

**Правило:** используй `Is` / `As`, кроме случаев когда уверен что не будет wrapping (например `io.EOF` в hot loop).

---

## Q74
### Wrapping ошибок: %w в fmt.Errorf

**`%w`** — добавляет контекст и сохраняет оригинальную ошибку для `errors.Is`/`As`.

```go
if err != nil {
    return fmt.Errorf("user service: get user %d: %w", id, err)
}
```

**`%v` vs `%w`:**
- `%v` — форматирование, теряет цепочку.
- `%w` — wrapping, сохраняет.

**Распаковка:**
```go
inner := errors.Unwrap(err)
```

**Несколько ошибок (Go 1.20+):**
```go
err := errors.Join(err1, err2, err3)
errors.Is(err, err1)  // true

// Или через множественный %w (Go 1.20+):
err := fmt.Errorf("oops: %w; %w", err1, err2)
```

**Best practice:**
- Добавляй контекст на каждом уровне.
- Не дублируй информацию (если функция называется `OpenFile`, не пиши «open file» в ошибке).

---

## Q75
### Custom error types

Свой тип ошибки = struct, реализующий `error`.

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}

// Использование:
return &ValidationError{Field: "email", Message: "invalid format"}

// Проверка:
var verr *ValidationError
if errors.As(err, &verr) {
    log.Printf("validation failed: %s", verr.Field)
}
```

**Поддержка wrapping:**
```go
type MyError struct {
    Op  string
    Err error
}

func (e *MyError) Error() string { return e.Op + ": " + e.Err.Error() }
func (e *MyError) Unwrap() error { return e.Err }  // ← позволяет errors.Is/As
```

---

## Q76
### Что такое recover? Как работает с panic?

**`recover()`** — встроенная функция, которая останавливает раскрутку стека при panic и возвращает значение, переданное в panic.

**Работает ТОЛЬКО внутри defer:**

```go
func safe() {
    defer func() {
        if r := recover(); r != nil {
            log.Printf("recovered: %v", r)
        }
    }()
    panic("boom")
}
```

**Правила:**
- `recover()` вне defer возвращает `nil`.
- Recover ловит только в **текущей** горутине.
- После recover функция нормально возвращается (или продолжает после deferred вызова).
- panic с `nil` (до Go 1.21) recover возвращал nil — не отличить от отсутствия panic. С 1.21 panic(nil) сама падает с `runtime.PanicNilError`.

**Применение:**
- HTTP middleware: панику в хендлере не должна валить весь сервер.
- Workers, которые должны переживать баги.

---

## Q77
### Когда panic уместен?

**Приемлемо:**
1. **Инициализация:** конфиг не загрузился — нет смысла продолжать.
2. **Невозможная ситуация** (assertion): `switch` где default не должен достигаться.
3. **API библиотек**: `regexp.MustCompile` (программист написал плохой regex).
4. **Mass-call API**: упрощение `Must*` функций.

**НЕ применяй:**
- Для обычных бизнес-ошибок.
- В библиотеках общего назначения.
- Для управления потоком (как goto/exception).

**Идиома `Must`:**
```go
func MustParse(s string) *URL {
    u, err := Parse(s)
    if err != nil {
        panic(err)
    }
    return u
}
```

---

## Q78
### Sentinel errors vs typed errors vs opaque errors

**Sentinel** — публичная переменная ошибки.
```go
var ErrNotFound = errors.New("not found")

if errors.Is(err, ErrNotFound) { ... }
```

Минус: добавляет в публичный API.

**Typed** — структурный тип, реализующий error.
```go
type NotFoundError struct{ ID int }
func (e *NotFoundError) Error() string { ... }

var nfe *NotFoundError
if errors.As(err, &nfe) { ... }
```

Плюс: возможность нести данные.

**Opaque** — ошибки скрываются за интерфейсами/функциями.
```go
// В пакете
type retryable interface { Retryable() bool }

func IsRetryable(err error) bool {
    var r retryable
    return errors.As(err, &r) && r.Retryable()
}
```

**Рекомендация:**
- Sentinel — для констант стандартных ошибок (`io.EOF`).
- Typed — когда нужны данные.
- Opaque — для библиотек, где не хочешь раскрывать детали.

---

## Q79
### Panic в горутине: что произойдёт?

**Panic в любой горутине без recover валит ВСЮ программу.**

```go
go func() {
    panic("oops")  // программа упадёт целиком!
}()
```

**Невозможно** поймать панику другой горутины из внешней — `recover` локален.

**Правильно — recover в КАЖДОЙ долгоживущей горутине:**
```go
func safeGo(name string, f func()) {
    go func() {
        defer func() {
            if r := recover(); r != nil {
                log.Printf("panic in %s: %v\n%s", name, r, debug.Stack())
            }
        }()
        f()
    }()
}
```

**В HTTP-сервере** `net/http` автоматически recovers панику в хендлере — не валит весь сервер. Но это исключение.

---

## Q80
### errgroup.Group: зачем нужен

`errgroup` (из `golang.org/x/sync/errgroup`) — это `WaitGroup` + сбор первой ошибки + автоматическая отмена через context.

```go
import "golang.org/x/sync/errgroup"

g, ctx := errgroup.WithContext(ctx)

for _, url := range urls {
    url := url
    g.Go(func() error {
        return fetch(ctx, url)
    })
}

if err := g.Wait(); err != nil {
    log.Print(err)
}
```

**Что делает:**
1. Запускает функции в горутинах.
2. Возвращает **первую non-nil ошибку**.
3. При первой ошибке `ctx` отменяется — остальные могут досрочно завершиться.
4. `g.Wait()` ждёт все.

**Go 1.20+: `SetLimit(n)`** для ограничения параллелизма:
```go
g.SetLimit(10)  // не больше 10 горутин одновременно
```

---

⬅️ [Интерфейсы](05-interfaces.md) · ➡️ [Runtime и GC](07-runtime-gc.md)
