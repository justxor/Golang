# 🔴 GC и runtime (Q81–Q90)

> Senior-уровень. Без понимания этих тем — не пройти собес на staff/principal.

---

## Q81
### Как работает GC в Go (tri-color mark-sweep)?

**Go GC = concurrent, non-generational, tri-color mark-sweep**.

**Tri-color алгоритм:**
- **Белый** — кандидат на удаление.
- **Серый** — обнаружен, но ещё не просканирован.
- **Чёрный** — просканирован, сохраняем.

**Фазы цикла:**

1. **Sweep termination** (STW, < 1 мс) — завершение предыдущего sweep.
2. **Mark setup** (STW, очень короткая) — включение write barrier.
3. **Marking** (concurrent) — обход графа от GC roots:
   - Стеки горутин.
   - Глобальные переменные.
   - Регистры.
   - Каждый достижимый объект помечается серым → чёрным.
4. **Mark termination** (STW, < 1 мс) — завершение mark, выключение write barrier.
5. **Sweep** (concurrent) — освобождение белых объектов в свободные блоки.

**Concurrent** = большая часть работы параллельна с приложением. STW паузы крошечные.

**Не generational:** в Go нет разделения young/old generation. Эксперименты были, но не дали выигрыша из-за escape analysis.

---

## Q82
### Что такое write barrier?

**Write barrier** — небольшая вставка кода в каждую операцию записи указателя во время GC. Нужна, чтобы concurrent marker не пропустил недавно созданные ссылки.

**Проблема:** мутатор (приложение) и маркер работают одновременно. Если приложение записывает указатель на белый объект в уже-чёрный объект — белый никогда не будет обойдён → удалится живой объект.

**Решение (Yuasa-style hybrid write barrier с Go 1.8):**
```pseudo
writePointer(slot, ptr):
    shade(*slot)  // помечаем серым старое значение
    shade(ptr)    // помечаем серым новое значение
    *slot = ptr
```

**Стоимость:** ~5-10% оверхеда во время GC цикла.

**Write barrier выключен** между GC циклами — там оверхеда нет.

---

## Q83
### Параметры GC: GOGC, GOMEMLIMIT

**`GOGC`** — соотношение между живой памятью и потребляемой до триггера GC. По умолчанию 100 (= 100%).

**Формула:** new heap = live * (1 + GOGC/100).

**Пример:** живых данных 100 MB, `GOGC=100` → следующий GC сработает при 200 MB heap.

**Настройки:**
```bash
GOGC=200 ./app   # реже GC, больше памяти
GOGC=50 ./app    # чаще GC, меньше памяти
GOGC=off ./app   # отключить (опасно!)
```

**`GOMEMLIMIT` (с Go 1.19):** жёсткий лимит памяти. GC начинает работать агрессивнее при приближении к нему.

```bash
GOMEMLIMIT=1GiB ./app
# Или программно:
debug.SetMemoryLimit(1 << 30)
```

**Production рекомендация:** установи `GOMEMLIMIT` чуть ниже container limit (например 90%) — спасает от OOMKill в k8s.

---

## Q84
### Stop-the-world паузы: сколько и почему?

**Современный Go (1.5+): STW паузы < 1 мс** даже на heap в десятки GB.

**Когда STW происходит:**
1. **Mark setup** — короткая остановка для включения write barrier и снимка корней.
2. **Mark termination** — короткая, для завершения mark.
3. **Sweep termination** — короткая, в начале нового цикла.

**Почему так мало:**
- Маркинг и sweep работают **параллельно** с приложением.
- Стеки горутин сканируются индивидуально.
- Используется write barrier вместо stop-the-world.

**Измерение:**
```bash
GODEBUG=gctrace=1 ./app
# выводит: gc 1 @0.001s 0%: 0.018+1.2+0.013 ms clock, ...
```

или в коде:
```go
var stats runtime.MemStats
runtime.ReadMemStats(&stats)
fmt.Println(stats.PauseTotalNs)
```

---

## Q85
### Что такое goroutine stack? Growable stacks

**Каждая горутина имеет свой стек.** Стартует с **2 KB (с Go 1.4)**, динамически растёт и **уменьшается**.

**Как растёт:**
1. Компилятор вставляет проверку в начале каждой функции: хватит ли стека для локальных переменных.
2. Если нет — runtime **аллоцирует новый стек в 2 раза больше**, копирует старый, обновляет указатели.
3. Старый стек освобождается.

**Максимальный размер:** 1 GB (по умолчанию). Можно изменить через `debug.SetMaxStack`.

**Stack shrinking:** во время GC, если стек используется на < 25% — runtime его сжимает (но не ниже 2 KB).

**Почему это круто:**
- Можно запустить **миллион горутин** на 16 GB RAM.
- Не нужно угадывать размер стека.

**Минус:** копирование стека = небольшой оверхед.

---

## Q86
### Profile-guided optimization (PGO)

**PGO (Go 1.21+)** — компилятор использует профиль выполнения для оптимизации горячих путей: inline, devirtualization.

**Workflow:**
```bash
# 1. Запусти приложение с pprof, собери профиль
curl http://localhost:6060/debug/pprof/profile?seconds=30 > cpu.pprof

# 2. Положи рядом с main.go как default.pgo
cp cpu.pprof default.pgo

# 3. Собери с PGO
go build -pgo=auto
# или: go build -pgo=./default.pgo
```

**Что улучшается:**
- Агрессивный inlining горячих функций.
- Devirtualization вызовов через интерфейсы.
- Лучшее register allocation.

**Эффект:** обычно 2-7% ускорения. На некоторых workloads до 14%.

---

## Q87
### pprof: какие профили бывают?

**`net/http/pprof` (или `runtime/pprof` для CLI):**

| Профиль | Что показывает | URL |
|---------|----------------|-----|
| **cpu** | Sample CPU usage | `/debug/pprof/profile?seconds=30` |
| **heap** | Heap allocations (live) | `/debug/pprof/heap` |
| **allocs** | Все аллокации с начала | `/debug/pprof/allocs` |
| **goroutine** | Стеки всех горутин | `/debug/pprof/goroutine` |
| **block** | Блокировки (sync) | `/debug/pprof/block` |
| **mutex** | Contention mutex | `/debug/pprof/mutex` |
| **threadcreate** | Создание OS-потоков | `/debug/pprof/threadcreate` |

**Использование:**
```bash
go tool pprof http://localhost:6060/debug/pprof/heap
(pprof) top10
(pprof) web    # граф
(pprof) list FuncName

# Flamegraph:
go tool pprof -http=:8080 cpu.pprof
```

**В коде:**
```go
import _ "net/http/pprof"

go http.ListenAndServe("localhost:6060", nil)
```

---

## Q88
### trace: что показывает?

**`runtime/trace`** — низкоуровневый профилировщик: переключения горутин, GC, syscalls, network — всё на тайм-линии.

```go
import "runtime/trace"

f, _ := os.Create("trace.out")
trace.Start(f)
defer trace.Stop()

// работа
```

**Просмотр:**
```bash
go tool trace trace.out
# Откроет браузер с UI
```

**Что увидишь:**
- График по `P` (логические CPU).
- Когда какая G выполняется.
- Длительность GC пауз.
- Goroutine blocking events.
- Syscall events.

**Когда полезно:**
- Долгие GC паузы.
- Неравномерная нагрузка на P.
- Долгий syscall блокирует прогресс.
- Поиск причины недогрузки CPU.

---

## Q89
### Как уменьшить аллокации?

**Шаги в порядке приоритета:**

**1. Профилировать (allocs):**
```bash
go test -bench=. -benchmem
go tool pprof -alloc_objects ./test allocs.pprof
```

**2. Pre-allocate slice/map:**
```go
// Плохо
var s []int
for i := 0; i < n; i++ { s = append(s, i) }

// Хорошо
s := make([]int, 0, n)
```

**3. Избегай interface{} boxing.** Используй generics.

**4. sync.Pool для повторно используемых объектов:**
```go
var bufPool = sync.Pool{New: func() any { return new(bytes.Buffer) }}
```

**5. Reduce escape to heap:**
- Возвращай значения, а не указатели на локальные.
- Не клади в interface{} (boxing).

**6. `strings.Builder` вместо `+`:**
```go
var b strings.Builder
for _, s := range parts { b.WriteString(s) }
result := b.String()
```

**7. Reuse buffers:**
```go
buf := make([]byte, 4096)
for {
    n, _ := r.Read(buf)  // переиспользуем
    process(buf[:n])
}
```

---

## Q90
### Inline функции в Go

**Inlining** — компилятор вставляет тело маленькой функции в место вызова. Убирает overhead вызова.

**Что инлайнится:**
- Маленькие функции (budget ~80 узлов AST).
- Без сложного control flow (defer, recover, range).
- Известные на compile-time.

**Что НЕ инлайнится:**
- Содержит `defer`, `recover`, `select`, `for range`.
- Вызовы через interface (без PGO).
- Слишком большие.
- Помечены `//go:noinline`.

**Проверка:**
```bash
go build -gcflags="-m" main.go
# Вывод: "can inline foo" или "cannot inline foo: function too complex"
```

**Помочь инлайнингу:**
- Разбивай сложные функции.
- Извлекай горячий путь в отдельную функцию.
- Используй PGO для девиртуализации.

**Не злоупотребляй:**
```go
//go:inline  // ❌ такого хинта в Go нет (в отличие от C)

//go:noinline  // ✅ только запрет, для бенчмарков
```

---

⬅️ [Errors](06-errors.md) · ➡️ [Production](08-production.md)
