# 🐹 Golang Interview Questions — 100 вопросов с разбором

> Профессиональный сборник вопросов с собеседований Go: от Junior до Senior. Каждый вопрос — с кратким разбором здесь и полным анализом в отдельном файле.

![Go](https://img.shields.io/badge/Go-1.23+-00ADD8?style=flat-square&logo=go) ![Level](https://img.shields.io/badge/level-Junior%20→%20Senior-blue?style=flat-square) ![Lang](https://img.shields.io/badge/lang-RU-red?style=flat-square) ![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)

---

## 📚 Как пользоваться

- 🟢 **Junior** — обязан знать на любом Go-собеседовании.
- 🟡 **Middle** — типовые вопросы для коммерческого опыта 1–3 года.
- 🔴 **Senior** — глубокое понимание runtime, GC, memory model.

Под каждым вопросом — **TL;DR** (короткий разбор) и ссылка **→ Полный разбор** на файл с кодом, таблицами, примерами и подводными камнями.

📂 Все файлы лежат в [`questions/`](questions/).

---

## 🟢 Основы языка (Q1–Q15) — [полный файл](questions/01-basics.md)

### Q1. Чем Go отличается от других языков? Главные особенности

**TL;DR:** Компилируемый, статически типизированный, со встроенной concurrency (goroutines + channels), GC, простой синтаксис, без классов/наследования (композиция через embedding), один бинарь без зависимостей, быстрая сборка.

→ [Полный разбор](questions/01-basics.md#q1)

### Q2. Какие типы данных есть в Go?

**TL;DR:** Базовые: bool, числовые (int/uint/float/complex), string, rune (int32), byte (uint8). Составные: array, slice, map, struct, channel, interface, pointer, function. Нет implicit conversion между числовыми типами.

→ [Полный разбор](questions/01-basics.md#q2)

### Q3. В чём разница между var, := и const?

**TL;DR:** `var` — явное объявление (можно без init, zero value). `:=` — short declaration, только внутри функций, тип выводится. `const` — неизменяемое значение времени компиляции (без указателей/слайсов/мап).

→ [Полный разбор](questions/01-basics.md#q3)

### Q4. Что такое zero value? Какие zero values у разных типов?

**TL;DR:** Значение по умолчанию для неинициализированной переменной: 0 для чисел, "" для string, false для bool, nil для pointer/slice/map/channel/interface/function, обнулённые поля у struct/array.

→ [Полный разбор](questions/01-basics.md#q4)

### Q5. Чем отличается string в кавычках от строки в backticks?

**TL;DR:** "..." — interpreted (escape-последовательности `\n`, `\t`). ``...`` — raw, символы как есть, многострочные, удобно для regex/JSON шаблонов.

→ [Полный разбор](questions/01-basics.md#q5)

### Q6. Как реализовано ООП в Go?

**TL;DR:** Без классов и наследования. Композиция через embedding, полиморфизм через интерфейсы (duck typing), инкапсуляция через регистр первой буквы (Public/private).

→ [Полный разбор](questions/01-basics.md#q6)

### Q7. Что такое struct? Чем отличается от class?

**TL;DR:** Struct — пользовательский тип-набор полей, без методов внутри тела. Методы навешиваются снаружи через receiver. Нет конструкторов, наследования, виртуальных методов.

→ [Полный разбор](questions/01-basics.md#q7)

### Q8. Что такое method receiver? Value vs Pointer receiver

**TL;DR:** Value receiver — копия структуры, безопасно для маленьких immutable. Pointer receiver — изменяет оригинал, дешевле для больших структур, нужен для мутаций. Правило: внутри одного типа — единообразно.

→ [Полный разбор](questions/01-basics.md#q8)

### Q9. Что такое iota и зачем оно нужно?

**TL;DR:** Генератор последовательных int-констант внутри const(). Используется для enum-подобных конструкций, битовых масок (`1 << iota`). Обнуляется в каждом const-блоке.

→ [Полный разбор](questions/01-basics.md#q9)

### Q10. Чем отличается new() от make()?

**TL;DR:** `new(T)` возвращает *T (нулевое значение). `make()` — только для slice/map/channel, инициализирует внутреннюю структуру, возвращает сам тип (не указатель).

→ [Полный разбор](questions/01-basics.md#q10)

### Q11. Что такое defer? В каком порядке выполняются?

**TL;DR:** Отложенный вызов до выхода из функции. Стек LIFO: последний defer — первый выполнится. Аргументы вычисляются сразу при объявлении, а не при выполнении.

→ [Полный разбор](questions/01-basics.md#q11)

### Q12. Что такое init() функция?

**TL;DR:** Спецфункция без аргументов, выполняется автоматически при импорте пакета (после переменных, до main). В одном пакете может быть несколько init(), порядок — по файлам алфавитно.

→ [Полный разбор](questions/01-basics.md#q12)

### Q13. Может ли функция возвращать несколько значений?

**TL;DR:** Да, идиоматично: `(value, error)`. Можно именовать возвращаемые значения (naked return). Часто используется паттерн "comma ok" для map/channel/type assertion.

→ [Полный разбор](questions/01-basics.md#q13)

### Q14. Что такое named return values?

**TL;DR:** Именованные возвращаемые значения в сигнатуре: `func f() (n int, err error)`. Автоматически объявляются как zero values, можно делать naked `return`. Полезно для defer + recover.

→ [Полный разбор](questions/01-basics.md#q14)

### Q15. Что такое variadic функции?

**TL;DR:** Функция с переменным числом аргументов через `...T`. Внутри — это слайс. При вызове можно передать существующий слайс через `slice...`. Пример: `fmt.Println`.

→ [Полный разбор](questions/01-basics.md#q15)

---

## 🟢🟡 Типы и память (Q16–Q25) — [полный файл](questions/02-types-memory.md)

### Q16. Где живут переменные: stack vs heap?

**TL;DR:** Решает компилятор через escape analysis. Stack — быстро, автоматически очищается. Heap — медленнее, требует GC. Указатель, который "убегает" из функции, отправляет объект на heap.

→ [Полный разбор](questions/02-types-memory.md#q16)

### Q17. Что такое escape analysis?

**TL;DR:** Анализ компилятора: можно ли разместить переменную на стеке. Если адрес "убегает" (возврат указателя, замыкание, interface{}, большой объект) — на heap. Проверка: `go build -gcflags="-m"`.

→ [Полный разбор](questions/02-types-memory.md#q17)

### Q18. Чем отличается указатель от значения?

**TL;DR:** Значение — самостоятельная копия данных. Указатель — адрес в памяти, `*T`, позволяет мутировать оригинал и избегать копирования. В Go нет арифметики указателей (без unsafe).

→ [Полный разбор](questions/02-types-memory.md#q18)

### Q19. Что такое nil pointer dereference?

**TL;DR:** Попытка разыменовать nil-указатель → panic. Возникает с интерфейсами, мапами, функциями, каналами. Защита: проверка `if x != nil` перед доступом.

→ [Полный разбор](questions/02-types-memory.md#q19)

### Q20. Чем отличается int от int32/int64?

**TL;DR:** `int` — платформозависимый (32 или 64 бита). `int32/int64` — фиксированный размер. Нельзя смешивать без явного преобразования. Для совместимости и сериализации — фиксированные размеры.

→ [Полный разбор](questions/02-types-memory.md#q20)

### Q21. Как работают строки внутри?

**TL;DR:** String = read-only struct { ptr *byte; len int }. Immutable. Срез строки — view (без копии). `len(s)` — байты, не руны. Для рун — `utf8.RuneCountInString`.

→ [Полный разбор](questions/02-types-memory.md#q21)

### Q22. Что такое rune и зачем оно нужно?

**TL;DR:** Alias для int32 — Unicode code point. Нужно для работы с не-ASCII текстом: `for i, r := range s` выдаёт руны. `len("ё")` = 2 (байты), но это 1 руна.

→ [Полный разбор](questions/02-types-memory.md#q22)

### Q23. Как сделать строку из []byte и наоборот?

**TL;DR:** `string(b)` и `[]byte(s)` — оба создают копию (string immutable). Для нулевых аллокаций — `unsafe` или `strings.Builder` / `bytes.Buffer`.

→ [Полный разбор](questions/02-types-memory.md#q23)

### Q24. Что такое type alias и type definition?

**TL;DR:** `type A = B` (alias) — то же самое, взаимозаменяемо. `type A B` (definition) — новый тип, нужен explicit cast, можно навешивать методы. Alias введён в Go 1.9.

→ [Полный разбор](questions/02-types-memory.md#q24)

### Q25. Что такое embedding структур?

**TL;DR:** Анонимное поле типа struct — поля и методы вложенного типа доступны напрямую (promoted). Не наследование — композиция. Можно "переопределить" метод, объявив на внешнем типе.

→ [Полный разбор](questions/02-types-memory.md#q25)

---

## 🟡 Slices и Maps (Q26–Q40) — [полный файл](questions/03-slices-maps.md)

### Q26. Как устроен slice внутри?

**TL;DR:** Структура из трёх полей: ptr (указатель на underlying array), len (длина), cap (ёмкость). Slice — header, копируется дёшево, но шарит массив.

→ [Полный разбор](questions/03-slices-maps.md#q26)

### Q27. В чём разница len() и cap()?

**TL;DR:** `len` — текущее число элементов. `cap` — сколько влезет без реаллокации. `append` аллоцирует новый массив, когда len > cap.

→ [Полный разбор](questions/03-slices-maps.md#q27)

### Q28. Как работает append?

**TL;DR:** Если cap хватает — добавляет на месте, len растёт. Если нет — выделяет новый массив (обычно ×2 до 1024, далее ×1.25), копирует, возвращает новый slice. Важно: всегда присваивать результат.

→ [Полный разбор](questions/03-slices-maps.md#q28)

### Q29. Что не так с этим кодом? (`s := s[:0]; append`)

**TL;DR:** Обнуление длины оставляет underlying array — старые данные доступны через cap. Утечка ссылок на объекты (GC не освободит). Решение: явный `clear()` (Go 1.21) или новый слайс.

→ [Полный разбор](questions/03-slices-maps.md#q29)

### Q30. Как сделать копию слайса?

**TL;DR:** `copy(dst, src)` — копирует min(len(dst), len(src)). Или `slices.Clone(s)` (Go 1.21). `b := a` — НЕ копия, разделяет underlying array.

→ [Полный разбор](questions/03-slices-maps.md#q30)

### Q31. Почему append может молча сломать данные?

**TL;DR:** Если cap родительского слайса позволяет — append модифицирует общий underlying array. Два слайса смотрят на одну память. Защита: `s[:n:n]` (full slice expression) или `slices.Clone`.

→ [Полный разбор](questions/03-slices-maps.md#q31)

### Q32. Как удалить элемент из slice?

**TL;DR:** `s = append(s[:i], s[i+1:]...)` (порядок сохраняется, O(n)). Или `s[i] = s[len(s)-1]; s = s[:len(s)-1]` (без порядка, O(1)). Или `slices.Delete` (Go 1.21).

→ [Полный разбор](questions/03-slices-maps.md#q32)

### Q33. Как устроена map внутри?

**TL;DR:** Hash table с bucket-ами (по 8 элементов). При collision — chain. При load factor > 6.5 — растёт ×2 (incremental rehash). Ключи должны быть comparable.

→ [Полный разбор](questions/03-slices-maps.md#q33)

### Q34. Безопасна ли map для конкурентного доступа?

**TL;DR:** НЕТ. Параллельная запись → fatal "concurrent map writes" (даже не recoverable panic). Решения: `sync.Mutex`, `sync.RWMutex`, `sync.Map`, sharded map.

→ [Полный разбор](questions/03-slices-maps.md#q34)

### Q35. Какой порядок итерации по map?

**TL;DR:** Случайный — Go специально рандомизирует, чтобы код не полагался на порядок. Для стабильности — собрать ключи, отсортировать (`slices.Sorted(maps.Keys(m))`).

→ [Полный разбор](questions/03-slices-maps.md#q35)

### Q36. Как проверить наличие ключа в map?

**TL;DR:** Comma-ok: `v, ok := m[key]`. Просто `v := m[key]` вернёт zero value (нельзя отличить отсутствие от нулевого значения).

→ [Полный разбор](questions/03-slices-maps.md#q36)

### Q37. Что такое sync.Map и когда его использовать?

**TL;DR:** Specialized concurrent map. Эффективна в двух паттернах: (1) ключи пишутся один раз, читаются много раз; (2) дизъюнктные ключи у разных горутин. В остальных случаях — Mutex+map быстрее.

→ [Полный разбор](questions/03-slices-maps.md#q37)

### Q38. Можно ли сравнивать slice/map напрямую?

**TL;DR:** НЕТ через ==. Только с nil. Для slice — `slices.Equal` / `reflect.DeepEqual`. Для map — `maps.Equal` (Go 1.21). reflect — медленно.

→ [Полный разбор](questions/03-slices-maps.md#q38)

### Q39. Что произойдёт при range по nil slice/map?

**TL;DR:** Ноль итераций, без panic. nil slice — `len=0, cap=0`, append работает. nil map — чтение возвращает zero value, запись → panic.

→ [Полный разбор](questions/03-slices-maps.md#q39)

### Q40. Что такое make([]int, 0, 100) и зачем?

**TL;DR:** Создаёт пустой слайс с pre-allocated capacity. Избегает реаллокаций при последующих append. Best practice когда финальный размер известен.

→ [Полный разбор](questions/03-slices-maps.md#q40)

---

## 🟡🔴 Concurrency: горутины, каналы, sync (Q41–Q60) — [полный файл](questions/04-concurrency.md)

### Q41. Что такое goroutine? Чем отличается от thread?

**TL;DR:** Легковесная функция, управляемая Go runtime. Стек начинается с ~2KB и растёт. Стоимость — микросекунды (vs миллисекунды у OS thread). Миллионы горутин на одну машину — норма.

→ [Полный разбор](questions/04-concurrency.md#q41)

### Q42. Что такое GMP-модель?

**TL;DR:** G (goroutine) — задача. M (machine) — OS-поток. P (processor) — логический CPU с локальной очередью G. Шедулер раскидывает G по P, выполняет на M. Количество P = `GOMAXPROCS`.

→ [Полный разбор](questions/04-concurrency.md#q42)

### Q43. Что такое work stealing?

**TL;DR:** Если у P закончились G в локальной очереди — он "ворует" половину из очереди другого P. Балансирует нагрузку без глобального локa, минимизирует idle CPU.

→ [Полный разбор](questions/04-concurrency.md#q43)

### Q44. Что такое channel? Buffered vs unbuffered

**TL;DR:** Типизированная очередь для коммуникации между горутинами. Unbuffered — синхронный rendezvous (отправитель ждёт получателя). Buffered — асинхронный, блокирует только когда буфер полный/пустой.

→ [Полный разбор](questions/04-concurrency.md#q44)

### Q45. Что произойдёт при отправке в закрытый канал?

**TL;DR:** Panic "send on closed channel". Чтение из закрытого канала — возвращает zero value с `ok=false`. Чтение из nil — блокировка навсегда. Закрывает всегда отправитель.

→ [Полный разбор](questions/04-concurrency.md#q45)

### Q46. Что такое select?

**TL;DR:** Конструкция для одновременного ожидания на нескольких channel-операциях. Выполняется один из готовых случаев, выбор случайный. `default` — non-blocking. `<-time.After` — таймаут.

→ [Полный разбор](questions/04-concurrency.md#q46)

### Q47. Чем отличается sync.Mutex от sync.RWMutex?

**TL;DR:** Mutex — эксклюзив. RWMutex — много читателей ИЛИ один писатель. RWMutex выгоден при reads >> writes. Иначе — overhead больше выигрыша. Не копировать после первого использования.

→ [Полный разбор](questions/04-concurrency.md#q47)

### Q48. Что такое sync.WaitGroup?

**TL;DR:** Счётчик горутин для ожидания их завершения. `Add(n)` до запуска, `Done()` в горутине (через defer), `Wait()` блокирует до нуля. Не копировать!

→ [Полный разбор](questions/04-concurrency.md#q48)

### Q49. Что такое sync.Once?

**TL;DR:** Гарантирует выполнение функции ровно один раз даже при конкурентных вызовах. Применение: lazy init, singleton, регистрация. Go 1.21: `sync.OnceValue/OnceFunc`.

→ [Полный разбор](questions/04-concurrency.md#q49)

### Q50. Что такое context.Context?

**TL;DR:** Стандартный способ отмены, дедлайнов и проброса request-scoped значений через границы API. Передаётся первым аргументом. Cancel освобождает ресурсы — обязательно `defer cancel()`.

→ [Полный разбор](questions/04-concurrency.md#q50)

### Q51. Когда использовать context.WithTimeout vs WithCancel?

**TL;DR:** WithCancel — ручная отмена. WithTimeout — отмена через duration. WithDeadline — отмена в фиксированный момент. Дочерний context отменяется при отмене родителя.

→ [Полный разбор](questions/04-concurrency.md#q51)

### Q52. Что такое race condition? Как ловить?

**TL;DR:** Несинхронизированный доступ к памяти из разных горутин. Минимум одна — запись. Поведение undefined. Детектор: `go test -race` / `go run -race`. Замедляет в 2–10 раз, только для тестов.

→ [Полный разбор](questions/04-concurrency.md#q52)

### Q53. Что такое deadlock и как избежать?

**TL;DR:** Все горутины ждут друг друга вечно. Runtime детектит "all goroutines are asleep". Правила: единый порядок захвата локов, таймауты, context-cancel, не звать чужой код под локом.

→ [Полный разбор](questions/04-concurrency.md#q53)

### Q54. Что такое goroutine leak?

**TL;DR:** Горутина заблокирована и никогда не завершится — память не освободится. Причины: чтение из канала, в который никто не пишет; забытый `cancel()`. Профилирование: pprof goroutine.

→ [Полный разбор](questions/04-concurrency.md#q54)

### Q55. Что такое fan-in / fan-out?

**TL;DR:** Fan-out — N воркеров читают из одного канала задач. Fan-in — слияние нескольких каналов в один. Базовые паттерны конкурентного пайплайна.

→ [Полный разбор](questions/04-concurrency.md#q55)

### Q56. Что такое errgroup?

**TL;DR:** `golang.org/x/sync/errgroup` — WaitGroup + первая ошибка + общий context. Идиоматично для параллельных запросов. `g.Go(fn)`, `g.Wait()`, `errgroup.WithContext`.

→ [Полный разбор](questions/04-concurrency.md#q56)

### Q57. Что такое semaphore-паттерн?

**TL;DR:** Buffered channel размера N как ограничитель параллелизма: занять — `sem <- struct{}{}`, освободить — `<-sem`. Или `golang.org/x/sync/semaphore`.

→ [Полный разбор](questions/04-concurrency.md#q57)

### Q58. Memory model: что гарантирует happens-before?

**TL;DR:** Go memory model описывает, какие записи видны при чтении. Гарантии дают: channel-операции, sync.Mutex, sync.Once, sync/atomic. Без синхронизации — порядок не определён.

→ [Полный разбор](questions/04-concurrency.md#q58)

### Q59. Когда использовать atomic вместо Mutex?

**TL;DR:** Atomic — простые операции над одной переменной (счётчик, флаг). Mutex — критическая секция с несколькими операциями. atomic быстрее, но требует понимания memory ordering. Go 1.19: `atomic.Int64` и др.

→ [Полный разбор](questions/04-concurrency.md#q59)

### Q60. Чем GOMAXPROCS отличается от runtime.NumCPU()?

**TL;DR:** NumCPU — число логических CPU. GOMAXPROCS — сколько может одновременно выполнять Go-кода (= N P). По умолчанию = NumCPU. В контейнерах часто нужно ставить вручную или uber-go/automaxprocs.

→ [Полный разбор](questions/04-concurrency.md#q60)

---

## 🟡 Интерфейсы и generics (Q61–Q70) — [полный файл](questions/05-interfaces.md)

### Q61. Что такое interface в Go?

**TL;DR:** Набор сигнатур методов. Тип удовлетворяет интерфейсу неявно (duck typing), без `implements`. Пустой интерфейс `interface{}` / `any` принимает любое значение.

→ [Полный разбор](questions/05-interfaces.md#q61)

### Q62. Как устроен interface внутри (iface, eface)?

**TL;DR:** Два указателя: type info + value. `eface` (empty) — *_type + unsafe.Pointer. `iface` (non-empty) — *itab + unsafe.Pointer. itab кеширует пару (interface, concrete type).

→ [Полный разбор](questions/05-interfaces.md#q62)

### Q63. Почему nil interface != interface с nil-указателем?

**TL;DR:** Interface равен nil только если оба компонента (тип и значение) nil. Если тип установлен, а значение nil — interface не nil. Классическая ошибка с `return err`.

→ [Полный разбор](questions/05-interfaces.md#q63)

### Q64. Что такое type assertion и type switch?

**TL;DR:** Assertion: `v, ok := x.(T)` — извлечь конкретный тип. Без ok — panic при несовпадении. Type switch: `switch v := x.(type) { case T: ... }` для разных типов.

→ [Полный разбор](questions/05-interfaces.md#q64)

### Q65. Когда использовать pointer receiver vs value receiver для interface?

**TL;DR:** Метод на pointer receiver удовлетворяет интерфейсу только для *T, не для T. Метод на value receiver — для обоих. Влияет на присваивание и встраивание.

→ [Полный разбор](questions/05-interfaces.md#q65)

### Q66. Что такое embedded interface?

**TL;DR:** Интерфейс может включать другие интерфейсы — методы объединяются. Пример: `io.ReadWriter` = Reader + Writer. Композиция малых интерфейсов в большие.

→ [Полный разбор](questions/05-interfaces.md#q66)

### Q67. Принцип "accept interfaces, return structs"

**TL;DR:** Функции принимают узкие интерфейсы (легче тестировать, мокать), но возвращают конкретные типы (вызывающий не теряет возможности). Малые интерфейсы — идиоматично.

→ [Полный разбор](questions/05-interfaces.md#q67)

### Q68. Что такое generics в Go 1.18+?

**TL;DR:** Параметризация типов: `func Map[T, U any](s []T, f func(T) U) []U`. Type constraints через интерфейсы. Compile-time monomorphization (GCShape). Не для всего — используйте, когда явно нужно.

→ [Полный разбор](questions/05-interfaces.md#q68)

### Q69. Что такое type constraint?

**TL;DR:** Интерфейс, описывающий допустимые типы для type parameter. `comparable`, `any`, union (`int | int64`), embedded constraints. Пакет `constraints`: Ordered, Integer и др.

→ [Полный разбор](questions/05-interfaces.md#q69)

### Q70. Когда НЕ стоит использовать generics?

**TL;DR:** Когда interface достаточно (поведение, не типы). Для одного-двух конкретных типов — проще написать руками. Generics дают читабельность для алгоритмов на коллекциях, не для бизнес-логики.

→ [Полный разбор](questions/05-interfaces.md#q70)

---

## 🟡 Ошибки, panic, recover (Q71–Q80) — [полный файл](questions/06-errors.md)

### Q71. Как принято обрабатывать ошибки в Go?

**TL;DR:** Явный возврат `(value, error)`, проверка `if err != nil`. Без try/catch. Ошибка — обычное значение, реализующее интерфейс `error` с методом `Error() string`.

→ [Полный разбор](questions/06-errors.md#q71)

### Q72. Что такое error wrapping (Go 1.13+)?

**TL;DR:** `fmt.Errorf("context: %w", err)` — оборачивает ошибку, сохраняя цепочку. Распаковка: `errors.Is(err, target)` (сравнение sentinel), `errors.As(err, &t)` (type match).

→ [Полный разбор](questions/06-errors.md#q72)

### Q73. errors.Is vs errors.As — в чём разница?

**TL;DR:** `Is` — равенство значению (sentinel ошибки: io.EOF, sql.ErrNoRows). `As` — извлечение определённого типа (custom error с дополнительными полями). Обходят всю цепочку wrap.

→ [Полный разбор](questions/06-errors.md#q73)

### Q74. Что такое sentinel errors?

**TL;DR:** Предопределённые переменные-ошибки уровня пакета: `var ErrNotFound = errors.New("not found")`. Сравниваются через `errors.Is`. Альтернатива — error types и behavioral interfaces.

→ [Полный разбор](questions/06-errors.md#q74)

### Q75. Custom error types: как правильно?

**TL;DR:** Struct, реализующий `Error() string`. Можно добавить `Unwrap()` для цепочки. Метод `Is(target error) bool` для кастомного матчинга. Указатель-получатель — для иммутабельности.

→ [Полный разбор](questions/06-errors.md#q75)

### Q76. Что такое panic и когда её использовать?

**TL;DR:** Аварийная остановка горутины с раскруткой стека. Использовать ТОЛЬКО для непоправимых ситуаций (битый инвариант, отсутствие критического ресурса при старте). Бизнес-ошибки → error.

→ [Полный разбор](questions/06-errors.md#q76)

### Q77. Что такое recover?

**TL;DR:** Останавливает panic. Работает только внутри defer. Возвращает значение из panic. После recover функция нормально возвращается. Применяется в HTTP-middleware, чтобы не убить весь процесс.

→ [Полный разбор](questions/06-errors.md#q77)

### Q78. Что произойдёт с горутиной при panic?

**TL;DR:** Если panic не пойман через recover в той же горутине — падает ВСЯ программа. recover в другой горутине НЕ ловит. Каждая горутина — свой recovery.

→ [Полный разбор](questions/06-errors.md#q78)

### Q79. Где defer + recover особенно важен?

**TL;DR:** HTTP-handlers, фоновые воркеры, gRPC. Шаблон: middleware с deferred recover, лог + 500-ответ. Не маскируйте баги — логируйте stack trace.

→ [Полный разбор](questions/06-errors.md#q79)

### Q80. Стоит ли возвращать panic из библиотеки?

**TL;DR:** Нет. Библиотека должна возвращать error. Panic уместен только внутри (например, нарушение инварианта). На границе API — конвертация в error.

→ [Полный разбор](questions/06-errors.md#q80)

---

## 🔴 Runtime, GC, профилирование (Q81–Q90) — [полный файл](questions/07-runtime-gc.md)

### Q81. Какой GC в Go и как он работает?

**TL;DR:** Concurrent tri-color mark-and-sweep с write barrier. Работает параллельно с приложением. Цель — STW < 1ms. Триггер — `GOGC` (по умолчанию 100% от живого heap).

→ [Полный разбор](questions/07-runtime-gc.md#q81)

### Q82. Что такое write barrier?

**TL;DR:** Хук на запись указателей во время GC mark phase. Гарантирует, что GC не пропустит достижимый объект, на который указатель появился во время сканирования.

→ [Полный разбор](questions/07-runtime-gc.md#q82)

### Q83. Что такое GOGC и GOMEMLIMIT?

**TL;DR:** `GOGC=100` — следующий GC при удвоении живого heap. `GOMEMLIMIT` (Go 1.19) — soft limit на общую память. В контейнерах — must, чтобы не выбило OOM-killer-ом.

→ [Полный разбор](questions/07-runtime-gc.md#q83)

### Q84. Что такое STW (Stop The World)?

**TL;DR:** Фазы GC, на время которых все горутины останавливаются. В Go — короткие (start mark, end mark). Цель runtime — субмиллисекунды. Измеряется в `runtime.MemStats`, tracing.

→ [Полный разбор](questions/07-runtime-gc.md#q84)

### Q85. Как профилировать Go-приложение?

**TL;DR:** `net/http/pprof` — CPU, heap, goroutine, mutex, block, allocs. Запуск: `go tool pprof` + flame graph. Бенчмарки: `go test -bench -cpuprofile -memprofile`.

→ [Полный разбор](questions/07-runtime-gc.md#q85)

### Q86. Что такое PGO (Profile-Guided Optimization)?

**TL;DR:** Go 1.21+: компилятор использует `default.pgo` (CPU-профиль prod-нагрузки) для inlining/devirtualization. Реальный выигрыш 2–14% CPU на горячих местах.

→ [Полный разбор](questions/07-runtime-gc.md#q86)

### Q87. Что такое trace и зачем он нужен?

**TL;DR:** `runtime/trace` — детальная картина шедулера, GC, syscalls, network. `go tool trace` открывает интерактивный viewer. Помогает найти латентность scheduler/sync.

→ [Полный разбор](questions/07-runtime-gc.md#q87)

### Q88. Как уменьшить число аллокаций?

**TL;DR:** sync.Pool для re-use, preallocate slices/maps, `strings.Builder` вместо конкатенации, escape-analysis-friendly код, избегать boxing в interface{}. Измерять `-benchmem`.

→ [Полный разбор](questions/07-runtime-gc.md#q88)

### Q89. Что такое sync.Pool?

**TL;DR:** Пул переиспользуемых объектов для снижения GC-нагрузки. Объекты могут быть удалены между GC циклами без предупреждения. Use cases: буферы, временные структуры.

→ [Полный разбор](questions/07-runtime-gc.md#q89)

### Q90. Что такое goroutine preemption?

**TL;DR:** До Go 1.14 — кооперативная (на вызовах функций). С Go 1.14 — asynchronous preemption через сигналы. Решает проблему "tight loop без syscall блокирует scheduler".

→ [Полный разбор](questions/07-runtime-gc.md#q90)

---

## 🔴 Production: тесты, паттерны, observability (Q91–Q100) — [полный файл](questions/08-production.md)

### Q91. Как организовать структуру Go-проекта?

**TL;DR:** Минимум: `cmd/<app>/main.go`, `internal/` (приватные пакеты), `pkg/` (публичные, опционально). Domain-driven, не layer-driven. Не складывайте всё в `/internal/utils`.

→ [Полный разбор](questions/08-production.md#q91)

### Q92. Как писать табличные тесты?

**TL;DR:** `tt := []struct{ name string; in T; want U }` + `t.Run(tt.name, ...)`. Параллелизация: `t.Parallel()`. С Go 1.22 — нет необходимости в `tt := tt`.

→ [Полный разбор](questions/08-production.md#q92)

### Q93. Что такое fuzzing в Go?

**TL;DR:** Go 1.18+: `func FuzzXxx(f *testing.F)` + `f.Add(seed)` + `f.Fuzz(func(t, input))`. Автогенерация входов, поиск panic / некорректного поведения. Запуск: `go test -fuzz=Fuzz`.

→ [Полный разбор](questions/08-production.md#q93)

### Q94. Как мокать зависимости?

**TL;DR:** Через интерфейсы на стороне потребителя. Инструменты: `gomock`, `testify/mock`, `counterfeiter`. Часто проще написать stub руками — проще, без кодогенерации.

→ [Полный разбор](questions/08-production.md#q94)

### Q95. graceful shutdown HTTP-сервера

**TL;DR:** `srv.Shutdown(ctx)` с таймаутом, перехват SIGINT/SIGTERM через `signal.NotifyContext`. Закрывать в обратном порядке инициализации: http → worker pool → БД.

→ [Полный разбор](questions/08-production.md#q95)

### Q96. Какой логгер использовать?

**TL;DR:** Стандартный `log/slog` (Go 1.21+) — structured, level-based, handlers. Раньше — zap/zerolog. Логи: JSON в prod, text в dev, всегда с context (request_id, user_id).

→ [Полный разбор](questions/08-production.md#q96)

### Q97. Observability: метрики, трассировка, логи

**TL;DR:** Метрики — Prometheus client. Трассировка — OpenTelemetry SDK. Логи — slog + correlation id. Дополнительно: pprof endpoint в проде (за auth), `expvar`.

→ [Полный разбор](questions/08-production.md#q97)

### Q98. Как организовать конфигурацию?

**TL;DR:** ENV-переменные как primary, fallback на flags или файлы. Пакеты: `envconfig`, `viper`. Валидация на старте — fail fast. Не хранить секреты в репо.

→ [Полный разбор](questions/08-production.md#q98)

### Q99. Database access: ORM vs sqlx vs database/sql

**TL;DR:** `database/sql` — низкоуровнево, контроль. `sqlx` — удобный mapping в struct. GORM — быстро, но магия и оверхэд. `sqlc` — codegen из SQL, типобезопасно.

→ [Полный разбор](questions/08-production.md#q99)

### Q100. Что нужно знать про deploy Go-приложения?

**TL;DR:** Single static binary (CGO_ENABLED=0), маленький multi-stage Dockerfile (FROM scratch / distroless), readiness/liveness probes, GOMAXPROCS под лимиты контейнера, GOMEMLIMIT.

→ [Полный разбор](questions/08-production.md#q100)

---

## 📦 Стек и версии

- **Go 1.21–1.23** — современные фичи: `slog`, `clear()`, `slices`/`maps`, `sync.OnceValue`, PGO, `GOMEMLIMIT`, loop var scoping (1.22).
- **Testing**: standard `testing`, `testify`, fuzzing, `-race`.
- **Tooling**: `go vet`, `staticcheck`, `golangci-lint`, `pprof`.

---

## 📡 Telegram-каналы для подготовки

Приоритетный порядок (бесплатные ресурсы):

1. **[@ai_machinelearning_big_data](https://t.me/ai_machinelearning_big_data)** — ML/AI/big data, прикладной Go.
2. **[@pythonl](https://t.me/pythonl)** — Python + общие practical posts (часто Go-сравнения).
3. **[Папка каналов](https://t.me/addlist/8vDUwYRGujRmZjFi)** — подборка для системных и backend разработчиков.

---

## 🎯 Как использовать перед собеседованием

- За 1–2 недели: пройти все 100 вопросов, разобраться где плаваете.
- За 3 дня: прогнать только 🔴 (Q41–Q60, Q81–Q100) — там обычно "режут".
- За день: бегло пройти TL;DR в этом README — освежить термины.

---

## 📜 Лицензия

MIT — используйте, дополняйте, делитесь.
