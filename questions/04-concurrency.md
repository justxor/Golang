# 🟡🔴 Goroutines и каналы (Q41–Q60)

> Главная тема Go-собесов на Middle+. Гарантированно спросят.

---

## Q41
### Что такое goroutine? Чем отличается от потока ОС?

**Goroutine** — лёгкий поток, управляемый Go runtime. Запускается через `go func()`.

| | Goroutine | OS Thread |
|---|-----------|-----------|
| Стек | 8 KB → растёт | 1-8 MB фиксированный |
| Переключение | User-space (~200 нс) | Kernel-space (~1-2 мкс) |
| Количество | Миллионы | Тысячи |
| Управление | Go runtime (M:N) | Ядро ОС |
| Создание | `go f()` | `pthread_create` |

**Принцип M:N:** Go мультиплексирует **M горутин на N потоков ОС**, где N = `GOMAXPROCS` (по умолчанию = число CPU).

```go
go func() {
    fmt.Println("Hi")
}()
time.Sleep(time.Millisecond)  // дать горутине отработать
```

---

## Q42
### Как работает планировщик Go (GMP-модель)?

Планировщик Go — **work-stealing scheduler** с моделью **GMP**:

- **G** (Goroutine) — задача.
- **M** (Machine) — поток ОС.
- **P** (Processor) — контекст выполнения (логический CPU). Количество = `GOMAXPROCS`.

**Алгоритм:**
1. Каждый `P` имеет **локальную очередь** горутин (LRQ, до 256).
2. Есть **глобальная очередь** (GRQ).
3. `M` берёт `P` и выполняет G из его LRQ.
4. Когда LRQ пуста — work stealing: половина у соседнего P.
5. Если ничего нет — берёт из GRQ или из netpoller.

**Preemption (с Go 1.14):** асинхронное вытеснение через SIGURG. До 1.14 — кооперативное.

**Goroutine блокируется (syscall, channel):** `M` отвязывается от `P`, освобождая `P` для других `M`. После анблокировки G попадает обратно в очередь.

---

## Q43
### Что такое M, P, G в планировщике?

Кратко (см. Q42):

- **G** = горутина. Структура с PC, SP, стек, статус.
- **M** = OS-поток, ассоциирован с одним `pthread`.
- **P** = логический процессор (контекст для исполнения), хранит LRQ.

**Инварианты:**
- `M` без `P` не может выполнять Go-код (только syscall).
- `P` без `M` не может выполняться вообще.
- Количество `P` = `GOMAXPROCS`, фиксировано.
- Количество `M` — динамическое (создаются по мере нужды).

**Просмотр изнутри:**
```go
runtime.NumCPU()         // ядра
runtime.NumGoroutine()   // активные G
runtime.GOMAXPROCS(0)    // P
```

---

## Q44
### Что такое канал (channel)?

**Канал** — типизированная **очередь FIFO** для синхронной/асинхронной коммуникации между горутинами. Принцип Go: *'Don't communicate by sharing memory; share memory by communicating.'*

```go
ch := make(chan int)        // небуферизованный
ch := make(chan int, 10)    // буферизованный, cap=10

ch <- 42       // отправка
v := <-ch      // приём
v, ok := <-ch  // ok=false если канал закрыт и пуст

close(ch)      // закрытие (только отправитель!)
```

**Направленные каналы:**
```go
func producer(out chan<- int) {}  // только запись
func consumer(in <-chan int)  {}  // только чтение
```

---

## Q45
### Буферизованный vs небуферизованный канал

**Небуферизованный (`cap=0`):** отправка и приём **синхронные** — отправитель блокируется, пока получатель не примет (rendezvous).

**Буферизованный (`cap>0`):** отправка не блокируется, пока буфер не заполнен. Приём не блокируется, пока буфер не пуст.

```go
// Небуферизованный — синхронизация
done := make(chan struct{})
go func() {
    work()
    done <- struct{}{}  // блокируется, пока main не прочитает
}()
<-done

// Буферизованный — fire-and-forget
logs := make(chan string, 100)
go logger(logs)
logs <- "msg"  // не блокируется, если буфер не полон
```

**Когда какой:**
- **Небуферизованный**: явная синхронизация, передача владения.
- **Буферизованный**: сглаживание burst-нагрузки, work pool.

**Правило:** размер буфера не от балды. Если не знаешь — небуферизованный.

---

## Q46
### Что произойдёт при чтении из закрытого канала?

**Чтение из закрытого канала:**
- Возвращает **zero value** немедленно, не блокируется.
- `v, ok := <-ch` → `ok == false`.

```go
ch := make(chan int, 2)
ch <- 1
ch <- 2
close(ch)

fmt.Println(<-ch)      // 1
fmt.Println(<-ch)      // 2
v, ok := <-ch
fmt.Println(v, ok)     // 0 false

// for range заканчивается, когда канал закрыт И пуст
for v := range ch {
    fmt.Println(v)
}
```

**Идиома:** закрытие = широковещательный сигнал «работа окончена».

---

## Q47
### Что произойдёт при записи в закрытый канал?

**Panic:**
```
panic: send on closed channel
```

**Правило:** закрывает только **отправитель**, и только когда уверен, что больше отправок не будет.

**Если у тебя несколько отправителей — координатор закрывает:**
```go
done := make(chan struct{})
var wg sync.WaitGroup

for i := 0; i < N; i++ {
    wg.Add(1)
    go func() {
        defer wg.Done()
        for {
            select {
            case <-done:
                return
            case ch <- work():
            }
        }
    }()
}

// Координатор:
close(done)
wg.Wait()
close(ch)  // безопасно: все отправители завершились
```

**Также panic:** `close()` уже закрытого канала.

---

## Q48
### Что такое select? Что такое default в select?

**`select`** — мультиплексирование каналов: ждёт первой готовой операции.

```go
select {
case v := <-ch1:
    use(v)
case ch2 <- x:
    sent()
case <-time.After(time.Second):
    timeout()
default:
    noActivity()
}
```

**Поведение:**
- Если готовых нет и нет `default` — блокируется.
- Если готовых несколько — выбирается **случайно**.
- `default` — выполняется немедленно если ничего не готово (non-blocking).

**Идиомы:**

**1. Non-blocking send:**
```go
select {
case ch <- v:
    // отправлено
default:
    // канал полон
}
```

**2. Timeout:**
```go
select {
case res := <-resultCh:
    handle(res)
case <-time.After(5 * time.Second):
    timeout()
}
```

**3. Cancel через context:**
```go
select {
case res := <-resultCh:
    return res
case <-ctx.Done():
    return ctx.Err()
}
```

---

## Q49
### Что такое sync.WaitGroup?

**Счётчик для ожидания** завершения N горутин.

```go
var wg sync.WaitGroup
for _, url := range urls {
    wg.Add(1)
    go func(u string) {
        defer wg.Done()
        fetch(u)
    }(url)
}
wg.Wait()
```

**Правила:**
1. `wg.Add(n)` вызывай **до** `go`, иначе race condition.
2. `wg.Done()` всегда через `defer`.
3. **Не копируй** `WaitGroup` — передавай по указателю.
4. `Wait()` можно вызывать только из одного места.

**Антипример:**
```go
go func() {
    wg.Add(1)  // ❌ race: Wait может выполниться раньше
    defer wg.Done()
    ...
}()
```

**Modern: `errgroup.Group`** — WaitGroup + сбор ошибок + cancellation.

---

## Q50
### Что такое sync.Mutex и sync.RWMutex?

**`sync.Mutex`** — взаимное исключение, эксклюзивный доступ.

```go
var mu sync.Mutex
mu.Lock()
defer mu.Unlock()
// критическая секция
```

**`sync.RWMutex`** — много читателей ИЛИ один писатель.
```go
var mu sync.RWMutex

mu.RLock()
v := data
mu.RUnlock()

mu.Lock()
data = newValue
mu.Unlock()
```

**Когда `RWMutex`:** читаем намного чаще, чем пишем (90/10+).

**Подводные камни:**
- **Не копировать** mutex — передавай по указателю или embed в struct.
- **Recursive lock = deadlock** — нельзя.
- **Defer Unlock** — обязательно (паника не освободит mutex иначе).
- `RWMutex` имеет больший overhead — для коротких секций обычный `Mutex` быстрее.

---

## Q51
### Что такое sync.Once?

Гарантирует, что переданная функция выполнится **ровно один раз**, даже при конкурентных вызовах.

```go
var (
    once     sync.Once
    instance *DB
)

func GetDB() *DB {
    once.Do(func() {
        instance = connect()
    })
    return instance
}
```

**Применение:** lazy initialization (синглтон без race), однократная регистрация.

**Внутри:** atomic + mutex — fast path через atomic CAS.

**Подводный камень:** если `f()` паникует — `Once` считает что выполнено, повторный `Do` не вызовет.

**Go 1.21+:** `sync.OnceFunc`, `sync.OnceValue`, `sync.OnceValues` — более удобные.
```go
getDB := sync.OnceValue(func() *DB { return connect() })
db := getDB()
```

---

## Q52
### Что такое sync.Pool? Зачем нужен?

**Pool временных объектов** — переиспользование вместо аллокаций. Снижает нагрузку на GC.

```go
var bufPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func handle() {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf)
    }()
    // ...
}
```

**Важно понимать:**
- Объекты могут быть **выкинуты GC** в любой момент (Pool — кэш, не хранилище).
- Не используй для соединений к БД (для этого есть `sql.DB`).
- Не клади указатели на pointer-fields, которые могут указывать на освобождённую память.
- `Get()` может вернуть случайный объект из пула — обязательно `Reset()`.

**Когда даёт буст:** короткоживущие большие объекты в hot path (buffer, json decoder).

---

## Q53
### Что такое context.Context?

**`context.Context`** — стандартный механизм для:
1. **Отмены** операций (cancellation).
2. **Таймаутов** и дедлайнов.
3. **Передачи request-scoped значений** (trace ID, user ID).

**Интерфейс:**
```go
type Context interface {
    Deadline() (time.Time, bool)
    Done() <-chan struct{}
    Err() error
    Value(key any) any
}
```

**Идиома:** первый параметр функции, `ctx context.Context`.

```go
func DoWork(ctx context.Context) error {
    select {
    case <-time.After(5 * time.Second):
        return nil
    case <-ctx.Done():
        return ctx.Err()  // context.Canceled или context.DeadlineExceeded
    }
}
```

---

## Q54
### Виды context: Background, TODO, WithCancel, WithTimeout, WithValue

```go
// Корни (никогда не отменяются)
ctx := context.Background()  // главный (main, init)
ctx := context.TODO()        // когда не знаешь что использовать (заглушка)

// Производные:
ctx, cancel := context.WithCancel(parent)
defer cancel()

ctx, cancel := context.WithTimeout(parent, 5*time.Second)
defer cancel()

ctx, cancel := context.WithDeadline(parent, time.Now().Add(time.Minute))
defer cancel()

ctx := context.WithValue(parent, "userID", 42)
// получение: userID := ctx.Value("userID").(int)
```

**Правила:**
1. **Всегда `defer cancel()`** — иначе утечка.
2. **Никогда не храни Context в struct** — передавай параметром.
3. **`WithValue` только для request-scoped данных**, не для optional параметров.
4. **Ключ для `WithValue` — собственный неэкспортируемый тип:**
```go
type ctxKey string
const userIDKey ctxKey = "userID"
ctx = context.WithValue(ctx, userIDKey, 42)
```

---

## Q55
### Что такое race condition? Как обнаружить?

**Race condition** — две или более горутин одновременно обращаются к одной памяти, и хотя бы одна из них пишет.

```go
var count int
for i := 0; i < 1000; i++ {
    go func() { count++ }()
}
// count != 1000 — гонка!
```

**Обнаружение — race detector:**
```bash
go run -race main.go
go test -race ./...
go build -race
```

Замедляет в 5-10 раз, увеличивает память в 5-10 раз. Использовать в CI и dev, не в проде.

**Решения:**
- `sync.Mutex` для shared state.
- `atomic` для счётчиков.
- Каналы для передачи владения.
- Иммутабельные данные.

**Atomic:**
```go
var count atomic.Int64
count.Add(1)
v := count.Load()
```

---

## Q56
### Что такое deadlock? Когда возникает?

**Deadlock** — все горутины заблокированы и ждут друг друга. Go runtime детектит и падает:
```
fatal error: all goroutines are asleep - deadlock!
```

**Классические причины:**

**1. Запись/чтение из небуферизованного канала без получателя/отправителя:**
```go
ch := make(chan int)
ch <- 1  // deadlock: некому читать
```

**2. Цикличная блокировка mutex:**
```go
// Горутина 1: lock A, потом B
// Горутина 2: lock B, потом A
// → deadlock
```

**3. Mutex заблокирован дважды одной горутиной (Go mutex НЕ реентрантный):**
```go
mu.Lock()
mu.Lock()  // вечная блокировка
```

**4. `wg.Wait()` без `wg.Done()`.**

**Профилактика:**
- Всегда блокируй mutexes в одном порядке.
- Таймауты через `select` + `context`.
- Закрытие каналов координатором.
- `defer mu.Unlock()`.

---

## Q57
### Memory model Go: happens-before

Go Memory Model определяет, когда **запись в одной горутине видна чтению в другой**.

**Гарантии happens-before:**
- Внутри одной горутины — порядок исходного кода.
- `go func()` запускается **после** оператора `go`.
- Завершение горутины не имеет happens-before с другими (нужна явная синхронизация).
- Send в канал **happens-before** соответствующего receive.
- Close канала **happens-before** receive, возвращающего zero value.
- `mu.Unlock()` **happens-before** следующего `mu.Lock()`.
- `sync.Once.Do(f)` завершение `f` **happens-before** возврата любого другого `Do`.
- `atomic` операции синхронизируются между собой.

**Без happens-before — UB:**
```go
var ready bool
var data int

// goroutine 1
data = 42
ready = true

// goroutine 2
if ready {
    fmt.Println(data)  // может напечатать 0!
}
```

**Правильно — atomic или mutex или channel.**

---

## Q58
### Паттерн fan-out / fan-in

**Fan-out:** одна задача → N воркеров.
**Fan-in:** N результатов → одна точка сбора.

```go
func fanOutFanIn(input <-chan Job, workers int) <-chan Result {
    out := make(chan Result)
    var wg sync.WaitGroup

    for i := 0; i < workers; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            for job := range input {
                out <- process(job)
            }
        }()
    }

    go func() {
        wg.Wait()
        close(out)
    }()

    return out
}
```

---

## Q59
### Паттерн pipeline

Цепочка стадий, соединённых каналами:

```go
func gen(nums ...int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for _, n := range nums { out <- n }
    }()
    return out
}

func sq(in <-chan int) <-chan int {
    out := make(chan int)
    go func() {
        defer close(out)
        for n := range in { out <- n * n }
    }()
    return out
}

// Использование:
for v := range sq(gen(1, 2, 3, 4)) {
    fmt.Println(v)  // 1 4 9 16
}
```

**Каждая стадия:**
- Получает из входного канала.
- Закрывает выходной по завершении.
- Работает до закрытия входного или сигнала done.

---

## Q60
### Утечка горутины: причины и как избежать

**Утечка** — горутина застряла навсегда (на канале, mutex, syscall), не освобождается → утечка памяти и стека.

**Причины:**

**1. Чтение из канала, в который никто не пишет:**
```go
ch := make(chan int)
go func() {
    v := <-ch  // навсегда, если никто не отправит
    use(v)
}()
```

**2. Запись в канал без получателя:**
```go
go func() {
    ch <- result  // блок, если получатель ушёл
}()
```

**3. Бесконечный for select без выхода:**
```go
go func() {
    for {
        select {
        case msg := <-ch:
            handle(msg)
        // нет case <-ctx.Done() — никогда не выйдет
        }
    }
}()
```

**Решения:**
- Всегда `<-ctx.Done()` в select.
- `defer close(ch)` у отправителя.
- `errgroup` с контекстом.

**Обнаружение:**
```go
fmt.Println(runtime.NumGoroutine())  // мониторинг

// или pprof
import _ "net/http/pprof"
// curl http://localhost:6060/debug/pprof/goroutine?debug=1
```

---

⬅️ [Slices и Maps](03-slices-maps.md) · ➡️ [Интерфейсы](05-interfaces.md)
