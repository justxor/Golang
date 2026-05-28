# 🟢 Основы языка (Q1–Q15)

> Базовые вопросы. Junior обязан знать все. Middle — отвечать без раздумий.

---

## Q1
### Чем Go отличается от других языков? Главные особенности

**Ответ:**

- **Компилируемый** в нативный машинный код (один статический бинарник, без VM).
- **Статическая типизация** с выводом типов (`:=`).
- **Сборщик мусора** (concurrent tri-color mark-sweep, паузы < 1 мс).
- **Встроенная concurrency**: goroutines (зелёные потоки) + channels.
- **Композиция вместо наследования**: нет классов, есть struct + embedding.
- **Минималистичный синтаксис**: 25 ключевых слов, один способ форматирования (gofmt).
- **Быстрая компиляция** (секунды на большой проект).
- **Богатая стандартная библиотека** (HTTP, JSON, crypto, testing — из коробки).

**Чего нет (намеренно):**
- Наследования классов.
- Generics до 1.18 (с 1.18 — есть, но ограниченные).
- Исключений (только error + panic).
- Перегрузки операторов и функций.
- Тернарного оператора.

**Где применяется:** backend микросервисы, CLI, инфраструктура (Docker, Kubernetes, Terraform), сетевые сервисы.

---

## Q2
### Какие типы данных есть в Go?

**Базовые (built-in):**

| Категория | Типы |
|-----------|------|
| **Булевы** | `bool` |
| **Целые знаковые** | `int8`, `int16`, `int32`, `int64`, `int` (платформозависимый) |
| **Целые беззнаковые** | `uint8` (=`byte`), `uint16`, `uint32`, `uint64`, `uint`, `uintptr` |
| **Вещественные** | `float32`, `float64` |
| **Комплексные** | `complex64`, `complex128` |
| **Строки** | `string` (immutable UTF-8) |
| **Руны** | `rune` (= `int32`, один unicode code point) |

**Составные:**
- `array` — фиксированный размер: `[5]int`.
- `slice` — динамический: `[]int`.
- `map` — хеш-таблица: `map[string]int`.
- `struct` — структура.
- `chan` — канал: `chan int`.
- `interface` — интерфейс.
- `func` — функциональный тип.
- `pointer` — указатель: `*int`.

---

## Q3
### В чём разница между `var`, `:=` и `const`?

```go
var x int = 10        // явное объявление с типом
var y = 10            // тип выводится (int)
var z int             // zero value = 0
a := 10               // short declaration (только внутри функций!)
const Pi = 3.14       // константа
const (
    StatusOK    = 200
    StatusError = 500
)
```

**Ключевые отличия:**

| | `var` | `:=` | `const` |
|---|-------|------|---------|
| Где можно | Везде | Только в функциях | Везде |
| Тип | Явный или выводится | Только выводится | Только базовые типы |
| Переопределение | `=` | `=` | Запрещено |
| Zero value | Да | Нет (нужна инициализация) | Нет |

**Подводный камень `:=`:**
```go
x, err := foo()
x, err := bar()  // ❌ ошибка: no new variables on left side
x, err = bar()   // ✅ если все уже объявлены — обычное присваивание
// НО:
x, newErr := bar()  // ✅ работает, если хотя бы одна переменная новая
```

---

## Q4
### Что такое zero value? Какие zero values у разных типов?

**Zero value** — значение по умолчанию для переменной, объявленной без инициализации. В Go **не бывает неинициализированных переменных** (в отличие от C).

| Тип | Zero value |
|-----|-----------|
| `bool` | `false` |
| `int`, `float`, `complex` | `0` |
| `string` | `""` (пустая строка) |
| `pointer`, `func`, `chan`, `map`, `slice`, `interface` | `nil` |
| `array` | каждый элемент = zero value своего типа |
| `struct` | каждое поле = zero value своего типа |

**Пример:**
```go
var s []int       // nil, но len(s)==0, cap(s)==0
var m map[string]int  // nil, читать можно, писать — panic
var p *int        // nil
var i interface{} // nil

type User struct {
    Name string
    Age  int
}
var u User  // u = User{Name: "", Age: 0}
```

**Идиоматичная проверка:** `if err != nil`, `if s == nil`.

---

## Q5
### Чем отличается строка в кавычках от строки в backticks?

```go
s1 := "hello\n"           // interpreted, \n = newline
s2 := `hello\n`           // raw, \n = два символа: \ и n
s3 := `Многострочная
строка без экранирования`
```

- **`"..."`** — *interpreted string literal*: обрабатывает escape-последовательности (`\n`, `\t`, `\u0041`).
- **`` `...` ``** — *raw string literal*: всё буквально, перенос строки сохраняется. Удобно для regex, JSON, SQL.

**Пример практический:**
```go
// Без raw — ад экранирования
regex := "\\d+\\.\\d+"
// С raw — читабельно
regex := `\d+\.\d+`
```

---

## Q6
### Как реализовано ООП в Go?

В Go **нет классов, нет наследования**. Есть три механизма, которые вместе закрывают потребность в ООП:

1. **Инкапсуляция** — через экспорт (большая буква = public, маленькая = private для пакета).
2. **Полиморфизм** — через интерфейсы (implicit).
3. **Композиция** — через embedding структур.

**Пример:**
```go
type Animal struct {
    Name string
}
func (a Animal) Speak() string { return "..." }

type Dog struct {
    Animal           // embedding (НЕ наследование!)
    Breed string
}

func (d Dog) Speak() string { return "Woof" }  // переопределение

d := Dog{Animal: Animal{Name: "Rex"}, Breed: "Labrador"}
fmt.Println(d.Name)     // "Rex" (поле от embedded)
fmt.Println(d.Speak())  // "Woof" (свой метод)
```

**Важно:** это не наследование, а **promotion** методов и полей. `Dog` НЕ является `Animal`, но имеет к нему доступ.

---

## Q7
### Что такое struct? Чем отличается от class?

**Struct** — набор именованных полей разных типов. Это просто данные.

```go
type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name"`
    Email string `json:"email"`
}

u := User{ID: 1, Name: "Bob", Email: "b@x.com"}
u2 := User{1, "Bob", "b@x.com"}  // positional (не рекомендуется)
```

**Отличия от class:**

| | Go struct | Class (Java/C++) |
|---|-----------|-------------------|
| Наследование | ❌ (только embedding) | ✅ |
| Конструктор | ❌ (конвенция: `NewUser()`) | ✅ |
| Деструктор | ❌ (GC + `defer`) | ✅ |
| Видимость | Только public/package | public/protected/private |
| Методы | Отдельно от struct | Внутри class |

**Идиома конструктора:**
```go
func NewUser(name string) *User {
    return &User{Name: name, ID: nextID()}
}
```

---

## Q8
### Что такое method receiver? Value vs Pointer receiver

```go
type Counter struct{ n int }

// Value receiver — копия
func (c Counter) GetValue() int { return c.n }

// Pointer receiver — указатель
func (c *Counter) Inc() { c.n++ }
```

**Когда использовать value receiver:**
- Маленькие неизменяемые структуры (`time.Time`).
- Когда метод не мутирует состояние.
- Когда нужна семантика value (immutable).

**Когда использовать pointer receiver:**
- Метод мутирует receiver.
- Структура большая (избежать копирования).
- Чтобы у всех методов был одинаковый receiver (консистентность).
- Когда struct содержит `sync.Mutex` — обязательно pointer!

**Подводный камень:**
```go
type Stringer interface { String() string }

func (c *Counter) String() string { return fmt.Sprint(c.n) }

var s Stringer = Counter{}     // ❌ не компилируется
var s Stringer = &Counter{}    // ✅
// Метод с pointer receiver требует адресуемый объект
```

---

## Q9
### Что такое iota и зачем оно нужно?

`iota` — встроенный счётчик в блоке `const`, который начинается с 0 и инкрементируется на каждой новой строке.

```go
const (
    Sunday    = iota  // 0
    Monday            // 1
    Tuesday           // 2
    Wednesday         // 3
)

// Битовые флаги:
const (
    Read    = 1 << iota  // 1
    Write                // 2
    Execute              // 4
)

// Единицы измерения:
const (
    _  = iota  // пропускаем 0
    KB = 1 << (10 * iota)  // 1024
    MB                      // 1048576
    GB
    TB
)
```

**Сбрасывается** на 0 при новом блоке `const (...)`.

---

## Q10
### Чем отличается `new()` от `make()`?

| | `new(T)` | `make(T, ...)` |
|---|----------|----------------|
| Что возвращает | `*T` (указатель) | `T` (значение) |
| Для каких типов | Любых | Только `slice`, `map`, `chan` |
| Инициализация | Zero value | Внутренние структуры готовы к работе |

```go
p := new(int)        // *int, *p = 0
s := make([]int, 5)  // []int с длиной 5, cap 5
m := make(map[string]int)
c := make(chan int, 10)  // буферизованный

// Эквиваленты:
p := new(int)
// то же самое, что:
var x int
p := &x
```

**Почему `make`, а не `new` для slice/map/chan?** Потому что внутри них есть служебные структуры (header у slice, бакеты у map), которые `new` оставит нулевыми, и работать с ними нельзя.

```go
var m map[string]int  // nil map
m["key"] = 1          // ❌ panic: assignment to entry in nil map

m := make(map[string]int)
m["key"] = 1          // ✅
```

---

## Q11
### Что такое defer? В каком порядке выполняются?

`defer` откладывает выполнение функции до момента **выхода из текущей функции** (return, panic, или end of function).

**LIFO** — Last In, First Out (стек):

```go
func main() {
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")
}
// Output:
// 3
// 2
// 1
```

**Главное использование:**
- Освобождение ресурсов: `defer file.Close()`, `defer mu.Unlock()`.
- Логирование выхода.
- Recover от panic.

**Подводные камни:**

1. **Аргументы вычисляются сразу:**
```go
i := 0
defer fmt.Println(i)  // напечатает 0, не 10
i = 10
```

2. **Защёлка в замыкании:**
```go
i := 0
defer func() { fmt.Println(i) }()  // напечатает 10
i = 10
```

3. **Defer в цикле = накопление:**
```go
for _, f := range files {
    f, _ := os.Open(f)
    defer f.Close()  // ❌ закроются только в конце функции
}
// Правильно — обернуть в анонимную функцию:
for _, name := range files {
    func() {
        f, _ := os.Open(name)
        defer f.Close()
        // работа
    }()
}
```

4. **Defer имеет накладные расходы** (с Go 1.14 минимальные, но всё же есть).

---

## Q12
### Что такое init() функция?

`init()` — специальная функция, которая выполняется **автоматически при импорте пакета**, до `main()`. Не имеет аргументов и возвращаемых значений.

```go
package db

var conn *sql.DB

func init() {
    conn = mustConnect()
}
```

**Правила:**
- Может быть **несколько** `init()` в одном пакете (даже в одном файле).
- Порядок: сначала **переменные пакета**, затем `init()` (по порядку файлов в алфавитном порядке).
- Сначала инициализируются **импортированные пакеты** (рекурсивно).
- В `main` пакете `init()` выполняется до `main()`.
- **Нельзя вызвать вручную.**

**Когда использовать:**
- Регистрация драйверов (`database/sql`, `image/png`).
- Чтение конфигурации.

**Когда НЕ использовать:**
- Когда инициализация может упасть с ошибкой — пользователь не сможет её обработать.
- Когда инициализация требует параметров.
- Тяжёлые вычисления — замедляют старт.

---

## Q13
### Может ли функция возвращать несколько значений?

Да, это идиома Go:

```go
func divmod(a, b int) (int, int) {
    return a / b, a % b
}

q, r := divmod(17, 5)  // 3, 2

// Игнорирование:
q, _ := divmod(17, 5)
```

**Главное использование:** возврат значения и ошибки.
```go
f, err := os.Open("file.txt")
if err != nil {
    return err
}
defer f.Close()
```

**Comma-ok идиомы:**
```go
v, ok := m[key]      // map: есть ли ключ
v, ok := <-ch        // channel: открыт ли
v, ok := i.(MyType)  // type assertion
```

---

## Q14
### Что такое named return values?

Можно дать имена возвращаемым значениям — они автоматически объявляются как переменные:

```go
func divide(a, b float64) (result float64, err error) {
    if b == 0 {
        err = errors.New("div by zero")
        return  // naked return — вернёт result=0, err=...
    }
    result = a / b
    return
}
```

**Плюсы:**
- Документация в сигнатуре.
- Удобно для `defer` (видны изменения в return value).

**Минусы:**
- Naked return на длинных функциях — нечитаемо.
- Может скрыть баги (забытое присваивание).

**Идиома для defer:**
```go
func process() (err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("panic: %v", r)
        }
    }()
    // ...
    return nil
}
```

---

## Q15
### Что такое variadic functions?

Функция, которая принимает переменное число аргументов одного типа. Обозначается `...T`.

```go
func sum(nums ...int) int {
    total := 0
    for _, n := range nums {
        total += n
    }
    return total
}

sum(1, 2, 3)        // 6
sum()               // 0

// Передача слайса:
nums := []int{1, 2, 3}
sum(nums...)        // распаковка
```

**Внутри функции** `nums` — обычный `[]int`.

**Подводный камень — модификация:**
```go
func modify(s ...int) {
    s[0] = 999
}

arr := []int{1, 2, 3}
modify(arr...)
fmt.Println(arr)  // [999 2 3] — slice указывает на тот же массив
```

**Только последний параметр** может быть variadic:
```go
func f(prefix string, vals ...int) {}  // ✅
func f(vals ...int, suffix string) {}  // ❌
```

**Известные примеры из stdlib:**
- `fmt.Println(args ...interface{})`
- `append(slice, elems...)`
- `strings.Join(s []string, sep string)` — НЕ variadic, обрати внимание.

---

⬅️ [Назад к README](../README.md) · ➡️ [Следующая категория: Типы и память](02-types-memory.md)
