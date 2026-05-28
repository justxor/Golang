# 🐹 Golang Interview Questions — 100 вопросов с разбором

> Профессиональный сборник вопросов с собеседований Go: от Junior до Senior. Каждый вопрос с подробным разбором, примерами кода и подводными камнями.

![Go](https://img.shields.io/badge/Go-1.23+-00ADD8?style=flat-square&logo=go) ![Level](https://img.shields.io/badge/level-Junior%20→%20Senior-blue?style=flat-square) ![Lang](https://img.shields.io/badge/lang-RU-red?style=flat-square) ![License](https://img.shields.io/badge/license-MIT-green?style=flat-square)

---

## 📚 Структура

| Категория | Файл | Вопросы |
|-----------|------|---------|
| **Основы языка** | [questions/01-basics.md](questions/01-basics.md) | Q1–Q15 |
| **Типы и память** | [questions/02-types-memory.md](questions/02-types-memory.md) | Q16–Q25 |
| **Slices и Maps** | [questions/03-slices-maps.md](questions/03-slices-maps.md) | Q26–Q40 |
| **Goroutines и каналы** | [questions/04-concurrency.md](questions/04-concurrency.md) | Q41–Q60 |
| **Интерфейсы** | [questions/05-interfaces.md](questions/05-interfaces.md) | Q61–Q70 |
| **Ошибки и panic** | [questions/06-errors.md](questions/06-errors.md) | Q71–Q80 |
| **GC и runtime** | [questions/07-runtime-gc.md](questions/07-runtime-gc.md) | Q81–Q90 |
| **Production и паттерны** | [questions/08-production.md](questions/08-production.md) | Q91–Q100 |

---

## 🎯 Уровни сложности

- 🟢 **Junior** — обязан знать на любом Go-собеседовании.
- 🟡 **Middle** — типовые вопросы для коммерческого опыта 1-3 года.
- 🔴 **Senior** — глубокое понимание runtime, GC, memory model.

---

## 📋 Полный индекс вопросов

### 🟢 Основы языка (Q1–Q15)

1. [Чем Go отличается от других языков? Главные особенности](questions/01-basics.md#q1)
2. [Какие типы данных есть в Go?](questions/01-basics.md#q2)
3. [В чём разница между var, := и const?](questions/01-basics.md#q3)
4. [Что такое zero value? Какие zero values у разных типов?](questions/01-basics.md#q4)
5. [Чем отличается string в кавычках от строки в backticks?](questions/01-basics.md#q5)
6. [Как реализовано ООП в Go?](questions/01-basics.md#q6)
7. [Что такое struct? Чем отличается от class?](questions/01-basics.md#q7)
8. [Что такое method receiver? Value vs Pointer receiver](questions/01-basics.md#q8)
9. [Что такое iota и зачем оно нужно?](questions/01-basics.md#q9)
10. [Чем отличается new() от make()?](questions/01-basics.md#q10)
11. [Что такое defer? В каком порядке выполняются?](questions/01-basics.md#q11)
12. [Что такое init() функция?](questions/01-basics.md#q12)
13. [Может ли функция возвращать несколько значений?](questions/01-basics.md#q13)
14. [Что такое named return values?](questions/01-basics.md#q14)
15. [Что такое variadic functions?](questions/01-basics.md#q15)

### 🟢🟡 Типы и память (Q16–Q25)

16. [Что такое pointer? Зачем нужны указатели в Go?](questions/02-types-memory.md#q16)
17. [Может ли быть указатель на указатель?](questions/02-types-memory.md#q17)
18. [Что такое escape analysis?](questions/02-types-memory.md#q18)
19. [Stack vs Heap: где хранятся переменные?](questions/02-types-memory.md#q19)
20. [Что такое struct tags? Зачем нужны?](questions/02-types-memory.md#q20)
21. [Что такое embedded struct (composition)?](questions/02-types-memory.md#q21)
22. [Как работает выравнивание (alignment) в Go?](questions/02-types-memory.md#q22)
23. [Что такое unsafe.Pointer? Когда использовать?](questions/02-types-memory.md#q23)
24. [Что такое рефлексия (reflect)?](questions/02-types-memory.md#q24)
25. [Какие минусы у рефлексии?](questions/02-types-memory.md#q25)

### 🟡 Slices и Maps (Q26–Q40)

26. [Что такое slice внутри? Из чего состоит?](questions/03-slices-maps.md#q26)
27. [Чем отличается array от slice?](questions/03-slices-maps.md#q26)
28. [Что такое len() и cap() у slice?](questions/03-slices-maps.md#q28)
29. [Как растёт slice при append?](questions/03-slices-maps.md#q29)
30. [Подводные камни append: shared underlying array](questions/03-slices-maps.md#q30)
31. [Как правильно копировать slice?](questions/03-slices-maps.md#q31)
32. [Как удалить элемент из slice?](questions/03-slices-maps.md#q32)
33. [Что такое nil slice vs empty slice?](questions/03-slices-maps.md#q33)
34. [Что такое map внутри? Hash table устройство](questions/03-slices-maps.md#q34)
35. [Почему map не потокобезопасна?](questions/03-slices-maps.md#q35)
36. [Как сделать concurrent-safe map?](questions/03-slices-maps.md#q36)
37. [Почему порядок обхода map случайный?](questions/03-slices-maps.md#q37)
38. [Как удалить элемент из map?](questions/03-slices-maps.md#q38)
39. [Что вернёт m[key] если ключа нет?](questions/03-slices-maps.md#q39)
40. [Comma-ok идиома для map](questions/03-slices-maps.md#q40)

### 🟡🔴 Goroutines и каналы (Q41–Q60)

41. [Что такое goroutine? Чем отличается от потока ОС?](questions/04-concurrency.md#q41)
42. [Как работает планировщик Go (GMP-модель)?](questions/04-concurrency.md#q42)
43. [Что такое M, P, G в планировщике?](questions/04-concurrency.md#q43)
44. [Что такое канал (channel)?](questions/04-concurrency.md#q44)
45. [Буферизованный vs небуферизованный канал](questions/04-concurrency.md#q45)
46. [Что произойдёт при чтении из закрытого канала?](questions/04-concurrency.md#q46)
47. [Что произойдёт при записи в закрытый канал?](questions/04-concurrency.md#q47)
48. [Что такое select? Что такое default в select?](questions/04-concurrency.md#q48)
49. [Что такое sync.WaitGroup?](questions/04-concurrency.md#q49)
50. [Что такое sync.Mutex и sync.RWMutex?](questions/04-concurrency.md#q50)
51. [Что такое sync.Once?](questions/04-concurrency.md#q51)
52. [Что такое sync.Pool? Зачем нужен?](questions/04-concurrency.md#q52)
53. [Что такое context.Context?](questions/04-concurrency.md#q53)
54. [Виды context: Background, TODO, WithCancel, WithTimeout, WithValue](questions/04-concurrency.md#q54)
55. [Что такое race condition? Как обнаружить?](questions/04-concurrency.md#q55)
56. [Что такое deadlock? Когда возникает?](questions/04-concurrency.md#q56)
57. [Memory model Go: happens-before](questions/04-concurrency.md#q57)
58. [Паттерн fan-out / fan-in](questions/04-concurrency.md#q58)
59. [Паттерн pipeline](questions/04-concurrency.md#q59)
60. [Утечка горутины: причины и как избежать](questions/04-concurrency.md#q60)

### 🟡 Интерфейсы (Q61–Q70)

61. [Что такое interface в Go?](questions/05-interfaces.md#q61)
62. [Как реализуются интерфейсы (implicit)?](questions/05-interfaces.md#q62)
63. [Что такое пустой интерфейс interface{} / any?](questions/05-interfaces.md#q63)
64. [Type assertion и type switch](questions/05-interfaces.md#q64)
65. [Внутреннее устройство interface (itab, iface, eface)](questions/05-interfaces.md#q65)
66. [Когда interface равен nil, а когда нет (typed nil)](questions/05-interfaces.md#q66)
67. [Принцип: «accept interfaces, return structs»](questions/05-interfaces.md#q67)
68. [Минимальные интерфейсы (io.Reader, io.Writer)](questions/05-interfaces.md#q68)
69. [Embedding interfaces](questions/05-interfaces.md#q69)
70. [Generics vs interfaces: когда что выбрать](questions/05-interfaces.md#q70)

### 🟡 Ошибки и panic (Q71–Q80)

71. [Как обрабатываются ошибки в Go?](questions/06-errors.md#q71)
72. [Чем отличается error от panic?](questions/06-errors.md#q72)
73. [errors.Is vs errors.As vs == ](questions/06-errors.md#q73)
74. [Wrapping ошибок: %w в fmt.Errorf](questions/06-errors.md#q74)
75. [Custom error types](questions/06-errors.md#q75)
76. [Что такое recover? Как работает с panic?](questions/06-errors.md#q76)
77. [Когда panic уместен?](questions/06-errors.md#q77)
78. [Sentinel errors vs typed errors vs opaque errors](questions/06-errors.md#q78)
79. [Panic в горутине: что произойдёт?](questions/06-errors.md#q79)
80. [errgroup.Group: зачем нужен](questions/06-errors.md#q80)

### 🔴 GC и runtime (Q81–Q90)

81. [Как работает GC в Go (tri-color mark-sweep)?](questions/07-runtime-gc.md#q81)
82. [Что такое write barrier?](questions/07-runtime-gc.md#q82)
83. [Параметры GC: GOGC, GOMEMLIMIT](questions/07-runtime-gc.md#q83)
84. [Stop-the-world паузы: сколько и почему?](questions/07-runtime-gc.md#q84)
85. [Что такое goroutine stack? Growable stacks](questions/07-runtime-gc.md#q85)
86. [Profile-guided optimization (PGO)](questions/07-runtime-gc.md#q86)
87. [pprof: какие профили бывают?](questions/07-runtime-gc.md#q87)
88. [trace: что показывает?](questions/07-runtime-gc.md#q88)
89. [Как уменьшить аллокации?](questions/07-runtime-gc.md#q89)
90. [Inline функции в Go](questions/07-runtime-gc.md#q90)

### 🔴 Production и паттерны (Q91–Q100)

91. [Структура проекта на Go (Standard Go Project Layout)](questions/08-production.md#q91)
92. [Go modules: go.mod, go.sum, vendoring](questions/08-production.md#q92)
93. [Graceful shutdown HTTP-сервера](questions/08-production.md#q93)
94. [Тестирование: table-driven tests, t.Run, t.Parallel](questions/08-production.md#q94)
95. [Mocking: интерфейсы + gomock / mockery](questions/08-production.md#q95)
96. [Бенчмарки: testing.B, b.ResetTimer, b.ReportAllocs](questions/08-production.md#q96)
97. [Fuzzing в Go](questions/08-production.md#q97)
98. [Generics: type parameters, constraints](questions/08-production.md#q98)
99. [Clean Architecture в Go](questions/08-production.md#q99)
100. [Observability: OpenTelemetry, structured logging slog](questions/08-production.md#q100)

---

## 🛠️ Как пользоваться

1. **Junior:** иди по порядку Q1 → Q40 (основы, типы, slices/maps).
2. **Middle:** упор на Q41–Q70 (concurrency, interfaces) + Q91–Q95.
3. **Senior:** обязательно Q57 (memory model), Q65 (itab/eface), Q81–Q90 (GC/runtime), Q98 (generics).
4. **Подготовка за вечер:** прочитай только разделы с пометкой 🔴.

---

## 📖 Источники

- **Официальное:** [go.dev/doc](https://go.dev/doc/), [Effective Go](https://go.dev/doc/effective_go), [Go Memory Model](https://go.dev/ref/mem).
- **Книги:** *The Go Programming Language* (Donovan/Kernighan), *Concurrency in Go* (Katherine Cox-Buday), *100 Go Mistakes* (Teiva Harsanyi).
- **Блоги:** [go.dev/blog](https://go.dev/blog), [Dave Cheney](https://dave.cheney.net/), [Bill Kennedy / Ardan Labs](https://www.ardanlabs.com/blog/).
- **Telegram:** [@ai_machinelearning_big_data](https://t.me/ai_machinelearning_big_data), [@pythonl](https://t.me/pythonl), [папка каналов](https://t.me/addlist/8vDUwYRGujRmZjFi).
- **Каналы Go:** [@Golang_google](https://t.me/Golang_google) — Go разработчики, разбор кода, гайды.
- **Практика:** [Exercism Go](https://exercism.org/tracks/go), [Gophercises](https://gophercises.com/).

---

## 📝 Лицензия

MIT — бери, форкай, улучшай. PR приветствуются.

> Обновлено: 2026 · Go 1.23+ · 100 вопросов · 8 категорий
