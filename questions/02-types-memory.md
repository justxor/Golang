# 🟢🟡 Типы и память (Q16–Q25)

> Указатели, escape analysis, рефлексия, выравнивание. Middle обязан понимать всё.

---

## Q16
### Что такое pointer? Зачем нужны указатели в Go?

**Pointer** — переменная, которая хранит адрес другой переменной в памяти.

```go
x := 42
p := &x          // p — указатель на x (*int)
fmt.Println(*p)  // 42 (разыменование)
*p = 100         // меняем x через указатель
fmt.Println(x)   // 100

var nilPtr *int  // zero value = nil
```

**Зачем нужны:**
1. **Изменить переданное значение** (без указателя — копия).
2. **Избежать копирования** больших структур.
3. **Различать «нет значения»** (nil) и «zero value».
4. **Pointer receiver** для методов.

**Чего нет в Go (в отличие от C/C++):**
- Арифметики указателей (`p+1` запрещено).
- Явного `malloc/free` — есть GC.

---

## Q17
### Может ли быть указатель на указатель?

Да, технически можно, но в идиоматичном Go встречается редко.

```go
x := 10
p := &x
pp := &p
fmt.Println(**pp)  // 10
```

**Где встречается на практике:**
- Когда функция должна изменить сам указатель (linked list, tree).
- В C-биндингах (CGo).

```go
func appendNode(head **Node, val int) {
    *head = &Node{Val: val, Next: *head}
}
```

---

## Q18
### Что такое escape analysis?

**Escape analysis** — анализ компилятора, который решает: разместить переменную **на стеке** (быстро, автоматическая очистка) или **на куче** (медленнее, нужен GC).

**Переменная "убегает" на кучу, если:**
1. Возвращается указатель на неё из функции.
2. Сохраняется в interface{} (boxing).
3. Передаётся в горутину.
4. Размер неизвестен на этапе компиляции (динамический slice).
5. Хранится в map/slice, который сам на куче.

```go
// На стеке:
func stackAlloc() int {
    x := 42
    return x
}

// На куче (escape):
func heapAlloc() *int {
    x := 42
    return &x  // указатель убегает
}
```

**Проверка:**
```bash
go build -gcflags="-m" main.go
# или подробнее:
go build -gcflags="-m -m" main.go
```

Вывод покажет: `moved to heap: x` или `x does not escape`.

---

## Q19
### Stack vs Heap: где хранятся переменные?

| | Stack | Heap |
|---|-------|------|
| Аллокация | Очень быстрая (двиг указателя) | Медленная (поиск свободного блока) |
| Деаллокация | Автоматически при выходе из функции | Через GC |
| Размер | Растёт динамически (8 KB → MB) | Ограничен GOMEMLIMIT/RAM |
| Доступ | Локальный для горутины | Общий |

**Каждая горутина имеет свой стек**, начинается с ~8 KB и растёт до 1 GB при необходимости.

**Где что:**
- Локальные переменные без escape — стек.
- `make([]int, 1000000)` — кеуча (размер большой).
- `interface{}{x}` — boxing значения на кучу.
- `goroutine func() { use(x) }` — `x` на куче.

**Важно:** в Go ты НЕ контролируешь явно, где аллоцировать — это решает компилятор. Можно только повлиять.

---

## Q20
### Что такое struct tags? Зачем нужны?

**Struct tag** — строка-метаданные после типа поля, доступная через reflect. Используется библиотеками для сериализации, валидации и т.п.

```go
type User struct {
    ID    int    `json:"id" db:"user_id" validate:"required"`
    Email string `json:"email" validate:"email"`
    Pass  string `json:"-"`  // не сериализуется
}
```

**Популярные библиотеки и их теги:**
- `encoding/json`: `json:"name,omitempty"`
- `database/sql` + sqlx: `db:"column_name"`
- `gorm`: `gorm:"primaryKey;autoIncrement"`
- `validator/v10`: `validate:"required,email"`
- `yaml.v3`: `yaml:"key"`

**Доступ через reflect:**
```go
t := reflect.TypeOf(User{})
f, _ := t.FieldByName("Email")
fmt.Println(f.Tag.Get("json"))     // "email"
fmt.Println(f.Tag.Get("validate")) // "email"
```

**Соглашение:** через пробел, `key:"value"`. Опции — через запятую.

---

## Q21
### Что такое embedded struct (composition)?

Когда поле структуры объявлено **без имени** — это embedding. Поля и методы внешней структуры «продвигаются» (promotion) во внешнюю.

```go
type Logger struct {
    Prefix string
}
func (l Logger) Log(msg string) { fmt.Println(l.Prefix, msg) }

type Service struct {
    Logger     // embedded
    Name string
}

s := Service{Logger: Logger{Prefix: "[svc]"}, Name: "api"}
s.Log("hi")        // promotion: вызов метода Logger
s.Logger.Log("hi") // явный доступ
s.Prefix           // promotion поля
```

**Это НЕ наследование:** `Service` НЕ является `Logger` (в смысле type assertion).

**Конфликт имён:**
```go
type A struct{ X int }
type B struct{ X int }
type C struct{ A; B }

c := C{}
c.X       // ❌ ambiguous selector
c.A.X     // ✅
```

**Полезно для:**
- Создания «декораторов» (`io.MultiWriter` композит `io.Writer`).
- Reuse без наследования.
- Mock-объектов в тестах.

---

## Q22
### Как работает выравнивание (alignment) в Go?

Поля структуры выравниваются по границам кратным размеру типа. Это влияет на размер struct.

```go
// Плохо: 24 байта на 64-bit
type Bad struct {
    a bool   // 1 байт + 7 padding
    b int64  // 8 байт
    c bool   // 1 байт + 7 padding
}

// Хорошо: 16 байт
type Good struct {
    b int64  // 8
    a bool   // 1
    c bool   // 1 + 6 padding
}
```

**Правило:** сортируй поля **от больших к маленьким**.

**Проверка размера:**
```go
unsafe.Sizeof(Bad{})   // 24
unsafe.Sizeof(Good{})  // 16
```

**Инструменты:**
- `fieldalignment` (golang.org/x/tools): `fieldalignment -fix ./...` — автоматически переставит поля.
- `go vet -shadow` для других проверок.

**Когда критично:** миллионы инстансов структуры (cache, queue) — экономия на размере = экономия RAM и cache miss.

---

## Q23
### Что такое unsafe.Pointer? Когда использовать?

`unsafe.Pointer` — указатель без типа. Позволяет:
1. Конвертировать между указателями разных типов.
2. Конвертировать в `uintptr` (адрес как число) и обратно.

```go
var i int32 = 42
p := unsafe.Pointer(&i)
f := (*float32)(p)         // re-interpret bytes как float32

// Доступ к полю по offset (НЕ ДЕЛАЙ ТАК БЕЗ КРАЙНЕЙ НУЖДЫ):
type S struct{ a, b int }
s := S{1, 2}
pb := unsafe.Pointer(uintptr(unsafe.Pointer(&s)) + unsafe.Offsetof(s.b))
```

**Когда оправдано:**
- Реализация низкоуровневых структур (atomic, mutex).
- CGo интероп.
- Zero-copy конверсии `string ↔ []byte` (опасно!).

**Опасности:**
- Обход системы типов — компилятор не поможет.
- GC может переместить объект, и `uintptr` устареет.
- Не работает с race detector.
- НЕ гарантирует совместимость между версиями Go.

**Правило:** если можешь обойтись без `unsafe` — обходись.

---

## Q24
### Что такое рефлексия (reflect)?

**Reflection** — возможность во время выполнения получать информацию о типе и значении переменной и модифицировать их.

```go
import "reflect"

x := 42
t := reflect.TypeOf(x)   // int
v := reflect.ValueOf(x)  // 42

fmt.Println(t.Kind())    // int
fmt.Println(v.Int())     // 42

// Изменение требует указатель:
p := &x
v2 := reflect.ValueOf(p).Elem()
v2.SetInt(100)
fmt.Println(x)           // 100
```

**Где используется в stdlib:**
- `encoding/json`, `encoding/xml` — маршалинг.
- `fmt.Printf` для `%v`.
- `database/sql` для сканирования в struct.

**Принцип:** `interface{}` хранит `(type, value)`. `reflect.TypeOf` и `reflect.ValueOf` извлекают эти части.

---

## Q25
### Какие минусы у рефлексии?

1. **Медленно** — в 10-100 раз медленнее статического кода (нет inlining, проверки типов в runtime).
2. **Нет проверок на этапе компиляции** — ошибки только в runtime, часто panic.
3. **Сложный код** — `reflect.Value.FieldByName(...).Set(...)` нечитаемо.
4. **Аллокации на куче** — все операции boxing.
5. **С generics (1.18+) часто не нужна.**

**Цитата Rob Pike:** *"Reflection is never clear."*

**Когда оправдано:**
- Универсальные сериализаторы (json, protobuf).
- Frameworks (ORM, DI, validators) — там без неё никак.
- Скрипты для разработчиков.

**Альтернативы:**
- Generics: `func Map[T, U any](s []T, f func(T) U) []U`.
- Code generation (`go generate` + шаблоны).
- Интерфейсы.

**Бенчмарк сравнения:**
```go
// reflect-маршалинг struct — ~500 ns/op
// сгенерированный код (easyjson) — ~80 ns/op
```

---

⬅️ [Основы](01-basics.md) · ➡️ [Slices и Maps](03-slices-maps.md)
