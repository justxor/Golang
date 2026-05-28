# 🔴 Production и паттерны (Q91–Q100)

> Senior-вопросы. Структура проекта, тесты, generics, observability.

---

## Q91
### Структура проекта на Go (Standard Go Project Layout)

**Нет официального стандарта**, но de-facto:

```
myapp/
├── cmd/                  # точки входа (main packages)
│   ├── api/main.go
│   └── worker/main.go
├── internal/             # код, видимый ТОЛЬКО внутри модуля
│   ├── domain/           # бизнес-сущности
│   ├── service/          # бизнес-логика
│   ├── repo/             # доступ к данным
│   └── transport/        # HTTP/gRPC хендлеры
├── pkg/                  # публичные пакеты (можно импортировать)
├── api/                  # OpenAPI/protobuf спецификации
├── configs/              # конфиги
├── deployments/          # k8s, docker-compose
├── scripts/              # CI/CD
├── go.mod
├── go.sum
└── README.md
```

**Главное:**
- **`internal/`** — особая папка: пакеты в ней импортируются только родительским модулем.
- **`cmd/<app>/main.go`** — один main на каждый бинарник.
- **Не клади всё в один пакет** — разделение по доменам, не по техническим уровням.

**Clean Architecture / Hexagonal:**
```
internal/
├── core/         # domain, use cases
├── ports/        # интерфейсы (Repository, EmailSender)
└── adapters/     # реализации (postgres, smtp)
```

---

## Q92
### Go modules: go.mod, go.sum, vendoring

**`go.mod`** — манифест модуля.
```
module github.com/org/myapp

go 1.23

require (
    github.com/jackc/pgx/v5 v5.5.0
    github.com/stretchr/testify v1.9.0
)

replace github.com/old/pkg => github.com/new/pkg v1.2.3
```

**`go.sum`** — хеши всех зависимостей (включая транзитивные). Проверяет целостность при сборке.

**Команды:**
```bash
go mod init github.com/org/myapp
go mod tidy             # синхронизация
go get github.com/x/y@latest
go get github.com/x/y@v1.2.3
go mod why -m all       # почему зависимость нужна
go mod graph            # дерево зависимостей
```

**Vendoring:**
```bash
go mod vendor           # копирует deps в ./vendor/
go build -mod=vendor    # использует vendor
```

**Когда vendor:**
- Air-gapped среды.
- Гарантия воспроизводимости даже если registry недоступен.
- Слой проверок (Snyk, Trivy сканируют vendor).

**Workspaces (Go 1.18+):**
```
go.work
use ./moduleA
use ./moduleB
```

---

## Q93
### Graceful shutdown HTTP-сервера

Корректное завершение: дать активным запросам докатиться до конца, не принимать новые.

```go
srv := &http.Server{Addr: ":8080", Handler: mux}

go func() {
    if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
        log.Fatal(err)
    }
}()

// Ждём сигнал
quit := make(chan os.Signal, 1)
signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
<-quit

log.Println("shutting down...")

ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err := srv.Shutdown(ctx); err != nil {
    log.Fatalf("forced shutdown: %v", err)
}
log.Println("server stopped")
```

**Что делает `Shutdown`:**
1. Закрывает listener.
2. Ждёт завершения активных запросов.
3. Если timeout — возвращает ошибку.

**Дополнительно:**
- Закрыть БД соединения, очереди.
- Дренировать рабочие очереди.
- В k8s — терпеть `preStop` hook + `terminationGracePeriodSeconds`.

---

## Q94
### Тестирование: table-driven tests, t.Run, t.Parallel

**Идиоматичный тест:**
```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive", 2, 3, 5},
        {"negative", -1, -1, -2},
        {"zero", 0, 0, 0},
    }

    for _, tt := range tests {
        tt := tt  // не нужно с Go 1.22+
        t.Run(tt.name, func(t *testing.T) {
            t.Parallel()
            got := Add(tt.a, tt.b)
            if got != tt.expected {
                t.Errorf("Add(%d,%d) = %d, want %d", tt.a, tt.b, got, tt.expected)
            }
        })
    }
}
```

**Особенности:**
- `t.Run(name, ...)` — subtests, фильтруются `-run=TestAdd/positive`.
- `t.Parallel()` — тест запустится параллельно с другими `Parallel`.
- **Go 1.22+:** loop variable scope изменился — `tt := tt` больше не нужно.

**Полезные функции:**
- `t.Helper()` — стек убирает helper из repor.
- `t.Cleanup(f)` — гарантированный cleanup после теста.
- `t.TempDir()` — временная папка, автоудаляется.
- `testing.Short()` — для пропуска долгих тестов в `-short`.

**testify:**
```go
import "github.com/stretchr/testify/require"

require.Equal(t, expected, got)
require.NoError(t, err)
```

---

## Q95
### Mocking: интерфейсы + gomock / mockery

**Подход:** код зависит от интерфейсов, тест подсовывает mock.

```go
// Production
type UserRepo interface {
    GetUser(ctx context.Context, id int) (*User, error)
}

type UserService struct {
    repo UserRepo
}

// Тест с ручным mock
type mockRepo struct {
    fn func(ctx context.Context, id int) (*User, error)
}
func (m *mockRepo) GetUser(ctx context.Context, id int) (*User, error) {
    return m.fn(ctx, id)
}

func TestUserService(t *testing.T) {
    repo := &mockRepo{
        fn: func(_ context.Context, id int) (*User, error) {
            return &User{ID: id, Name: "Bob"}, nil
        },
    }
    svc := &UserService{repo: repo}
    // ...
}
```

**Генераторы:**

**mockery** (вариант 1, де-факто стандарт):
```bash
mockery --name=UserRepo --dir=./internal
```

**gomock** (от Google):
```bash
go install go.uber.org/mock/mockgen@latest
mockgen -source=repo.go -destination=mocks/repo_mock.go
```

```go
ctrl := gomock.NewController(t)
defer ctrl.Finish()
mock := mocks.NewMockUserRepo(ctrl)
mock.EXPECT().GetUser(gomock.Any(), 1).Return(&User{}, nil)
```

---

## Q96
### Бенчмарки: testing.B, b.ResetTimer, b.ReportAllocs

```go
func BenchmarkFib(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Fib(20)
    }
}
```

**Запуск:**
```bash
go test -bench=. -benchmem -benchtime=5s
go test -bench=BenchmarkFib -cpuprofile=cpu.out
```

**Полезные методы:**
```go
func BenchmarkParse(b *testing.B) {
    data := loadFixture()  // подготовка
    b.ResetTimer()         // не учитывать подготовку
    b.ReportAllocs()       // показать allocs/op

    for i := 0; i < b.N; i++ {
        Parse(data)
    }

    b.StopTimer()
    cleanup()
}
```

**Параллельный бенчмарк:**
```go
func BenchmarkParallel(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            Work()
        }
    })
}
```

**Сравнение версий:**
```bash
go test -bench=. -count=10 > old.txt
# изменения в коде
go test -bench=. -count=10 > new.txt
benchstat old.txt new.txt
```

---

## Q97
### Fuzzing в Go

**С Go 1.18+** в stdlib. Генерирует случайные входы, ищет краши и баги.

```go
func FuzzParse(f *testing.F) {
    f.Add("hello world")  // seed
    f.Add("")

    f.Fuzz(func(t *testing.T, s string) {
        result, err := Parse(s)
        if err != nil {
            return
        }
        // инварианты: результат должен пройти эти проверки
        if !utf8.ValidString(result) {
            t.Errorf("Parse returned invalid UTF-8: %q", result)
        }
    })
}
```

**Запуск:**
```bash
go test -fuzz=FuzzParse -fuzztime=30s
```

**Когда падает:**
- Crash-input сохраняется в `testdata/fuzz/FuzzParse/`.
- Этот файл становится regression-тестом — `go test` его пропускает.

**Где полезно:**
- Парсеры (JSON, URL, regex).
- Кодировщики/декодировщики.
- Stateful protocols.

---

## Q98
### Generics: type parameters, constraints

**С Go 1.18:** функции и типы могут принимать параметры-типы.

```go
// Тип параметра T, ограничен интерфейсом any
func Map[T, U any](s []T, f func(T) U) []U {
    r := make([]U, len(s))
    for i, v := range s { r[i] = f(v) }
    return r
}

// Использование
doubled := Map([]int{1,2,3}, func(x int) int { return x*2 })
```

**Constraints (ограничения типов):**
```go
// Из stdlib (cmp пакет)
type Ordered interface {
    ~int | ~int8 | ~int16 | ~int32 | ~int64 |
    ~uint | ~uint8 | ~uint16 | ~uint32 | ~uint64 |
    ~float32 | ~float64 | ~string
}

func Max[T Ordered](a, b T) T {
    if a > b { return a }
    return b
}
```

**`~int`** — type set: любой тип, чей underlying — int (включая `type MyInt int`).

**Generic struct:**
```go
type Stack[T any] struct { items []T }

func (s *Stack[T]) Push(v T)  { s.items = append(s.items, v) }
func (s *Stack[T]) Pop() (T, bool) {
    if len(s.items) == 0 { var zero T; return zero, false }
    v := s.items[len(s.items)-1]
    s.items = s.items[:len(s.items)-1]
    return v, true
}

s := &Stack[int]{}
s.Push(1)
```

**Когда НЕ нужно:** если интерфейс решает задачу так же — оставь интерфейс.

---

## Q99
### Clean Architecture в Go

**Принцип:** зависимости направлены внутрь — к domain layer. Внешние слои зависят от внутренних, не наоборот.

**Слои (снаружи внутрь):**
1. **Frameworks & Drivers** — HTTP, gRPC, БД-драйверы.
2. **Interface Adapters** — handlers, repository implementations.
3. **Use Cases (Application)** — бизнес-сценарии.
4. **Entities (Domain)** — чистые модели и правила.

**Структура:**
```
internal/
├── domain/
│   ├── user.go              # type User struct
│   └── errors.go            # var ErrNotFound = ...
├── usecase/
│   ├── user_service.go      # бизнес-логика
│   └── ports.go             # type UserRepo interface
└── infra/
    ├── http/                # handlers
    └── postgres/
        └── user_repo.go     # реализует UserRepo
```

**Domain не знает про БД и HTTP** — только бизнес-сущности.

**Use case** принимает интерфейс `UserRepo`, реализация подставляется в `main.go` через DI:
```go
func main() {
    db := postgres.New(cfg.DSN)
    repo := postgres.NewUserRepo(db)
    svc := usecase.NewUserService(repo)
    handler := http.NewHandler(svc)
    http.ListenAndServe(":8080", handler)
}
```

**Тестирование:** use case тестируется с mock-реализацией repo.

---

## Q100
### Observability: OpenTelemetry, structured logging slog

**Три столпа:** **logs, metrics, traces**. Все через OpenTelemetry.

**slog (Go 1.21+) — structured logging:**
```go
import "log/slog"

logger := slog.New(slog.NewJSONHandler(os.Stdout, nil))
slog.SetDefault(logger)

slog.Info("user logged in",
    "user_id", 42,
    "ip", "1.2.3.4",
)
// {"time":"...", "level":"INFO", "msg":"user logged in", "user_id":42, "ip":"1.2.3.4"}
```

**OpenTelemetry (traces + metrics):**
```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/trace"
)

tracer := otel.Tracer("user-service")

func GetUser(ctx context.Context, id int) (*User, error) {
    ctx, span := tracer.Start(ctx, "GetUser")
    defer span.End()

    span.SetAttributes(attribute.Int("user.id", id))

    u, err := repo.Get(ctx, id)
    if err != nil {
        span.RecordError(err)
        return nil, err
    }
    return u, nil
}
```

**Метрики (Prometheus / OTel):**
```go
meter := otel.Meter("user-service")
requestCount, _ := meter.Int64Counter("http.requests.total")
requestCount.Add(ctx, 1, metric.WithAttributes(
    attribute.String("method", "GET"),
    attribute.Int("status", 200),
))
```

**Production стек:**
- Logs → Loki / OpenSearch.
- Metrics → Prometheus / Mimir.
- Traces → Tempo / Jaeger.
- Dashboards → Grafana.

**Correlation:** trace_id в логах = можно перейти от лога к трейсу:
```go
logger.InfoContext(ctx, "processing", "trace_id", trace.SpanFromContext(ctx).SpanContext().TraceID())
```

---

🎉 **Поздравляю — это все 100 вопросов!**

⬅️ [Runtime и GC](07-runtime-gc.md) · 🏠 [К началу](../README.md)
