# 🟡 Интерфейсы (Q61–Q70)

> Полиморфизм в Go. Любимая тема Senior-собесов.

---

## Q61
### Что такое interface в Go?

**Interface** — тип, описывающий **набор методов**. Любой тип, реализующий эти методы, удовлетворяет интерфейсу.

```go
type Stringer interface {
    String() string
}

type Color struct{ R, G, B int }

func (c Color) String() string {
    return fmt.Sprintf("#%02X%02X%02X", c.R, c.G, c.B)
}

var s Stringer = Color{255, 0, 0}
fmt.Println(s)  // #FF0000
```

**Интерфейс — это:**
- Контракт поведения.
- Способ полиморфизма.
- Точка расширения для тестов (mocking).

**В Go интерфейсы:**
- Реализуются **неявно** (нет `implements`).
- Маленькие (1-3 метода) — идиоматично.
- Объявляются на стороне **потребителя**, не производителя.

---

## Q62
### Как реализуются интерфейсы (implicit)?

**Implicit (structural) implementation** — если у типа есть все нужные методы, он автоматически удовлетворяет интерфейсу. Никаких `implements`.

```go
// Интерфейс объявлен в пакете io
type Writer interface {
    Write(p []byte) (n int, err error)
}

// Мой тип НЕ знает про io.Writer
type Counter struct{ n int }

func (c *Counter) Write(p []byte) (int, error) {
    c.n += len(p)
    return len(p), nil
}

// Автоматически удовлетворяет io.Writer
var w io.Writer = &Counter{}
```

**Плюсы:**
- Можно адаптировать чужие типы под свои интерфейсы.
- Не нужно менять источник, чтобы он соответствовал интерфейсу.
- Маленькие фокусные интерфейсы.

**Минусы:**
- Не сразу видно, какие интерфейсы реализует тип.
- Можно сломать совместимость, изменив сигнатуру метода.

**Compile-time проверка:**
```go
var _ io.Writer = (*Counter)(nil)  // assertion в виде blank identifier
```

---

## Q63
### Что такое пустой интерфейс interface{} / any?

**`interface{}`** — интерфейс без методов. Любой тип ему удовлетворяет (как `Object` в Java или `Any` в Kotlin).

**`any`** — алиас для `interface{}` начиная с Go 1.18. Идиоматично с 1.18.

```go
var x any = 42
var y any = "hello"
var z any = struct{ N int }{N: 1}

func PrintAny(v any) {
    fmt.Println(v)
}
```

**Где применяется:**
- `fmt.Println(args ...any)`.
- Контейнеры до generics.
- JSON: `map[string]any`.

**Минусы:**
- Теряем типизацию.
- Boxing → аллокация на куче.
- Нужен type assertion для использования.

**Generics часто заменяют `any`:**
```go
// До 1.18
func First(s []any) any { return s[0] }

// С 1.18+
func First[T any](s []T) T { return s[0] }
```

---

## Q64
### Type assertion и type switch

**Type assertion** — извлечение конкретного типа из интерфейса.

```go
var i any = "hello"

s := i.(string)        // panic если не string
s, ok := i.(string)    // безопасная форма
if ok {
    fmt.Println(s)
}
```

**Type switch** — несколько проверок типа подряд.

```go
func describe(i any) string {
    switch v := i.(type) {
    case int:
        return fmt.Sprintf("int: %d", v)
    case string:
        return fmt.Sprintf("string: %q", v)
    case []byte:
        return fmt.Sprintf("bytes: %x", v)
    case nil:
        return "nil"
    default:
        return fmt.Sprintf("unknown: %T", v)
    }
}
```

**Подводные камни:**
- Type switch на интерфейс — учти, что nil тоже case.
- `case A, B:` — `v` будет иметь тип исходного `i`, не A или B.

---

## Q65
### Внутреннее устройство interface (itab, iface, eface)

**Интерфейс внутри = 2 слова (16 байт на 64-bit):**

**Для интерфейса БЕЗ методов (`any`, `interface{}`) — `eface`:**
```go
type eface struct {
    _type *_type        // указатель на runtime-описание типа
    data  unsafe.Pointer  // указатель на значение
}
```

**Для интерфейса С методами — `iface`:**
```go
type iface struct {
    tab  *itab           // method table
    data unsafe.Pointer
}

type itab struct {
    inter *interfacetype   // тип интерфейса
    _type *_type           // конкретный тип
    hash  uint32
    _     [4]byte
    fun   [1]uintptr       // указатели на методы (variable size)
}
```

**itab кэшируется** (по паре (interface, concrete type)) — повторное создание дешёвое.

**Вызов метода через интерфейс:**
1. Достать `tab.fun[i]` (индекс метода).
2. Вызвать с `data` как receiver.

Это дороже прямого вызова (~2-3 ns vs 0.3 ns), но дешевле reflect.

---

## Q66
### Когда interface равен nil, а когда нет (typed nil)

**Классическая ловушка Go:**

```go
type MyError struct{}
func (e *MyError) Error() string { return "boom" }

func mayFail() error {
    var err *MyError = nil
    return err  // ⚠️
}

err := mayFail()
fmt.Println(err == nil)  // false!
```

**Почему:** интерфейс хранит `(type, value)`. Здесь:
- `type = *MyError` (НЕ nil)
- `value = nil`

Интерфейс равен `nil` только если **обе части nil**.

**Правильно:**
```go
func mayFail() error {
    var err *MyError
    if somethingWrong {
        err = &MyError{}
    }
    if err != nil {
        return err
    }
    return nil  // явный typed nil
}
```

**`errcheck` и `go vet` помогают** ловить такие баги.

---

## Q67
### Принцип: «accept interfaces, return structs»

**Идиома Go:** функции **принимают интерфейсы**, **возвращают конкретные типы**.

```go
// ❌ Плохо
func NewLogger() Logger { ... }

// ✅ Хорошо
func NewLogger() *FileLogger { ... }

// Принимающие функции:
func Process(r io.Reader) {}
func Save(w io.Writer) {}
```

**Почему:**
- **Возврат struct** — пользователь получает все методы, может приводить к нужному интерфейсу.
- **Возврат интерфейса** — теряем функциональность, скрываем тип, усложняем эволюцию API.
- **Приём интерфейса** — функция работает с любой реализацией, легко тестируется.

**Исключения:** factory функции в DI-фреймворках, абстракции (database/sql.Driver).

---

## Q68
### Минимальные интерфейсы (io.Reader, io.Writer)

**Принцип:** *"The bigger the interface, the weaker the abstraction"* — Rob Pike.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type Closer interface { Close() error }

// Композиция:
type ReadCloser interface {
    Reader
    Closer
}
```

**Преимущества:**
- Тесная композиция через embedding.
- Просто реализовать (1 метод).
- Адаптеры легко пишутся (`io.LimitReader`, `io.MultiWriter`).

**Антипример:**
```go
type UserService interface {
    GetUser(id int) (*User, error)
    CreateUser(u *User) error
    DeleteUser(id int) error
    UpdateUser(u *User) error
    ListUsers() ([]*User, error)
    SearchUsers(q string) ([]*User, error)
    // ... 20 методов
}
// Не интерфейс, а класс. Лучше разделить.
```

---

## Q69
### Embedding interfaces

Интерфейс может содержать другие интерфейсы:

```go
type ReadWriter interface {
    Reader  // получает Read()
    Writer  // получает Write()
}

// Эквивалентно:
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}
```

**Применение в stdlib:**
- `io.ReadCloser`, `io.WriteCloser`, `io.ReadWriteCloser`.
- `sort.Interface` композируется в `sort.StringSlice`.

**Расширение существующего:**
```go
type LoggedReader interface {
    io.Reader
    Logger() *slog.Logger
}
```

---

## Q70
### Generics vs interfaces: когда что выбрать

**До 1.18:** интерфейсы для всего.
**С 1.18+:** generics для типобезопасных контейнеров и алгоритмов.

**Когда interface:**
- Полиморфизм поведения (разные реализации).
- Mocking в тестах.
- Подмена реализации (storage, logger).
- Когда тип не имеет значения, важен контракт.

**Когда generics:**
- Контейнеры (`Stack[T]`, `Set[T]`).
- Алгоритмы (`Min[T]`, `Map[T, U]`, `Filter[T]`).
- Когда хочешь сохранить статическую типизацию.
- Когда хочешь избежать boxing/unboxing.

```go
// Generics — типобезопасно
func Map[T, U any](s []T, f func(T) U) []U {
    r := make([]U, len(s))
    for i, v := range s {
        r[i] = f(v)
    }
    return r
}

// Interface — расширяемость
type Storage interface {
    Get(id string) ([]byte, error)
    Put(id string, data []byte) error
}
```

**Type constraints в generics:**
```go
type Number interface {
    ~int | ~int64 | ~float64
}

func Sum[T Number](s []T) T {
    var total T
    for _, v := range s { total += v }
    return total
}
```

`~int` — type set: любой тип, чей underlying type = int.

---

⬅️ [Concurrency](04-concurrency.md) · ➡️ [Ошибки и panic](06-errors.md)
