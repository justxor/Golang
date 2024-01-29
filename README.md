# Golang
golang interview questions

Вопросы с собееседований Golang 
https://t.me/Golang_google - телкграм канал для Go разработчиков, полезные инструменты, разбор кода, гайды и уроки.

@Golang_google
---------------------------------------------------------------
package main

import "fmt"

func main(){
    fmt.Println("Hello, world")
}

main()  // go run файл.go
Как реализовано ООП в Go?
 
Какие типы данных есть в Go?
var str string = "hello"
var str2 string = `Hello, 
I am multiline string`

str
hello
str := "hello"
str
hello
numbers := [...]int{0, 1, 2}
numbers 
[0 1 2]
var numbers = [3]int{}
numbers 
[0 0 0]
our_map := map[string]int {"1": 11, "2": 22}

our_map
map[1:11 2:22]
Что такое рефлексия в go
и чем она полезна?
package main

import (
    "fmt"
    "reflect"
)

func main() {
    x := []int{0, 0, 0}
    y := 12
    fmt.Println("Тип x:", reflect.TypeOf(x))
    fmt.Println("Тип y:", reflect.TypeOf(y))
}

main()  // go run название_файла.go 
Тип x: []int
Тип y: int
package main

import (
    "fmt"
    "reflect"
)

func main() {
    x := 42
    v := reflect.ValueOf(&x).Elem() // Получаем reflect.Value

    fmt.Println("Исходное значение x:", x)
    v.SetInt(44) // Изменяем значение x
    fmt.Println("Новое значение x:", x)
}

main()  // go run название_файла.go
Исходное значение x: 42
Новое значение x: 44
Что из себя представляют
числовые константы в Go?
var pi float64 = 3.14  // переменные можно менять, а const нет
pi = 3
pi
3
const pi float64 = 3.14
pi
pi = 3
cannot assign to a const: pi <float64>
const (
    age int = 30
    capacity int = 100
)
const (
    daysInWeek  = 7
    hoursInDay  = 24
    minutesInHour = 60
)
Что такое канал, и какие виды
каналов бывают в Go?
package main

import "fmt"

func main() {
    var c chan int  // канал со значением <nil>
    fmt.Println(c)

    var intCh chan int = make(chan int)  // небуферизированный
    chanBuf := make(chan bool, 3)  // буферизированный канал
    fmt.Println(intCh, chanBuf)
} 

main()
<nil>
0xc0000a2a20 0xc0006c62a0
Как работают буферизованные
и небуферизованные каналы?
package main

import (
	"fmt"
)

func main() {
	numbers := make(chan int, 5)  
	// канал numbers не может хранить более пяти целых чисел — это буферный канал с емкостью 5
	counter := 10
	for i := 0; i < counter; i++ {
		select {
		// здесь происходит обработка
		case numbers <- i * i:
			fmt.Println("About to process", i)
		default:
			fmt.Print("No space for ", i, " ")
		}
    // мы начинаем помещать данные в numbers, однако когда канал заполнен, он перестанет сохранять данные и будет выполняться ветка default
	}
	fmt.Println()
	for {
		select {
		case num := <-numbers:
			fmt.Print("*", num, " ")
		default:
			fmt.Println("Nothing left to read!")
			return
		}
	}
}


main()
About to process 0
About to process 1
About to process 2
About to process 3
About to process 4
No space for 5 No space for 6 No space for 7 No space for 8 No space for 9 
*0 *1 *4 *9 *16 Nothing left to read!
Можно ли в Go закрыть канал
со стороны читателя?
func main() {
    dataCh := make(chan int)
    stopCh := make(chan struct{})
    
    go func() {
        for {
            select {
            case data, ok := <-dataCh:
                if !ok {
                    // Канал закрыт, прекращаем обработку
                    return
                }
                // Обработка данных
                fmt.Println(data)
            case <-stopCh:
                // Получен сигнал остановки, закрываем канал dataCh
                close(dataCh)
                return
            }
        }
    }()

    // Отправка данных в канал
    dataCh <- 1
    dataCh <- 2

    // Отправка сигнала остановки
    stopCh <- struct{}{}
}


main()
1
2
Расскажи про строки в Go
var s string = "Привет"
s
Привет
s := "hello"
s
hello
runes := []rune(s)
runes
[1055 1088 1080 1074 1077 1090]
runes := []rune{'П', 'р', 'и', 'в', 'е', 'т'}
s := string(runes)
s
Привет
len(s)  // len - это объём в байтах (не кол-во символов)
12
Как эффективно конкатенировать
множество строк?
// 1 способ через +=

func concat1(values []string) string {
	s := ""
	for _, value := range values {
		s += value
	}
	return s
}

concat1([]string{"a", "b", "c", "d"})
abcd
// 2 способ через структуру Builder

import "strings"

func concat2(values []string) string {
	sb := strings.Builder{}
	for _, value := range values {
		_, _ = sb.WriteString(value) // добавляется строка
	}
	return sb.String()
}

concat2([]string{"a", "b", "c", "d"})
abcd
// 3 способ

func concat3(values []string) string {
	total := 0
	for i := 0; i < len(values); i++ { // проводятся итерации по каждой строке для вычисления общего числа байтов
		total += len(values[i])
	}
	sb := strings.Builder{}
	sb.Grow(total) // вызывается Grow с аргументом, равным этому общему числу
	for _, value := range values {
		_, _ = sb.WriteString(value)
	}
	return sb.String()
}

concat3([]string{"a", "b", "c", "d"})
abcd
package main

import (
    "time"
    "fmt"
)

func main() {
    s := make([]string, 10000, 10000)
    
	startTime := time.Now()
	concat1(s)
	fmt.Println("Время concat1:", time.Since(startTime))

	startTime := time.Now()
	concat2(s)
	fmt.Println("Время concat2:", time.Since(startTime))

	startTime := time.Now()
	concat3(s)
	fmt.Println("Время concat3:", time.Since(startTime))
}

main()
Время concat1: 1.4938ms
Время concat2: 20.2723ms
Время concat3: 41.4777ms
Что из себя представляет стабы (stubs)
и моки (mock) в контексте тестирования?
 
Что делает runtime.newobject()?
 
repl.go:14:7: not a type: runtime.newobject <*ast.SelectorExpr>
Какие численные типы есть в Go?
package main

import (
	"fmt"
	"reflect"
)

func main() {
	var_int := 10
	var_float64 := 1.2
	var_complex128 := complex(2, 3)
	r, im := real(var_complex128), imag(var_complex128)

	fmt.Println(reflect.TypeOf(var_int),
            	reflect.TypeOf(var_float64),
            	reflect.TypeOf(var_complex128),
            	reflect.TypeOf(r), reflect.TypeOf(im))
}

main()
int float64 complex128 float64 float64
var a int64 = 1289302
reflect.TypeOf(a)
int64
Что такое обычный int и какие есть
ньюансы его реализации?
a := 4

fmt.Println(a / 0)
repl.go:3:17: division by zero
import "strconv"

strconv.Atoi("4")
4 <nil>
strconv.Itoa(4)
4
Какая проблема в этом коде?
var counter int

for i := 0; i < 1000; i++ {
   go func() {  // каждая итерация запускается в отдельной go
      counter++
   }()
}

counter
409
package main

import "sync"

func main() {
    var counter int
    var mutex sync.Mutex
    
    for i := 0; i < 1000; i++ {
       go func() {
          mutex.Lock()  // блокируем выполнение других go
          counter++
          mutex.Unlock()
       }()
    }
}    
Converter.Type(): unsupported types.Type: *types.TypeParam
Как проверить тип переменной
в среде выполнения?
package main

import "fmt"

func do(i interface{}) {
  switch v := i.(type) {
    case int:
      fmt.Printf("Double %v is %v\n", v, v*2)
    case string:
      fmt.Printf("%q is %v bytes long\n", v, len(v))
    default:
      fmt.Printf("I don't know type %T!\n", v)
  }
}

func main() {
  do(21)
  do("hello")
  do(true)
}

main()
Double 21 is 42
"hello" is 5 bytes long
I don't know type bool!
// а вообще для этого лучше использовать reflect.TypeOf()

import "reflect"

x := "hello"
reflect.TypeOf(x)
string
Как выполнить несколько условий
в одном операторе switch case?
x := 1

switch x {
    case 1, 2, 3:
        fmt.Println("x is 1, 2, or 3")
    case 4, 5, 6:
        fmt.Println("x is 4, 5, or 6")
    default:
        fmt.Println("x is not in any of the above cases")
}
x is 1, 2, or 3
x := 2

switch x {
    case 1:
        fmt.Println("x is 1")
        fallthrough
    case 2:
        fmt.Println("x is 1 or 2")
        fallthrough
    case 3:
        fmt.Println("x is 3")
    default:
        fmt.Println("x is not in any of the above cases")
}
x is 1 or 2
x is 3
Что такое heap и stack?
 
Где выделяется память под переменную?
Можно ли этим управлять?
 
Что такое указатель на указатель в Go?
package main

import "fmt"

func main() {
    a := 50
    var b *int = &a  // b — указатель на переменную a
    var c **int = &b // c — указатель на указатель b

    fmt.Println("Значение a:", a)   // Исходное значение
    fmt.Println("Адрес a:", &a)     // Адрес переменной a
    fmt.Println("Значение b:", b)   // Адрес, хранящийся в b (адрес a)
    fmt.Println("Разыменование b:", *b) // Разыменование b (значение a)
    fmt.Println("Значение c:", c)   // Адрес, хранящийся в c (адрес b)
    fmt.Println("Разыменование c:", *c) // Разыменование c (значение b, т.е. адрес a)
    fmt.Println("Двойное разыменование c:", **c) // Двойное разыменование c (значение a)
}


main()
Значение a: 50
Адрес a: 0xc0000897b8
Значение b: 0xc0000897b8
Разыменование b: 50
Значение c: 0xc00032b200
Разыменование c: 0xc0000897b8
Двойное разыменование c: 50
Реализовать структуру данных "стек"
с методами pop, append и top.
package main

type Stack struct {
  items []int
}

func (s *Stack) Push(data int) {
  s.items = append(s.items, data)
}

func (s *Stack) Pop() {
  if s.IsEmpty() {
    return
  }
  s.items = s.items[:len(s.items)-1]
}

func (s *Stack) Top() (int, error) {
  if s.IsEmpty() {
    return 0, fmt.Errorf("stack is empty")
  }
  return s.items[len(s.items)-1], nil
}

func (s *Stack) IsEmpty() bool {
  if len(s.items) == 0 {
    return true
  }
  return false
}

func (s *Stack) Print() {
  for _, item := range s.items {
    fmt.Print(item, " ")
  }
  fmt.Println()
}
repl.go:12:6: not a package: "s" in s.IsEmpty <*ast.SelectorExpr>
Как ведут себя срезы в Go
на граничных значениях?
slice := make([]int, 3, 3)  // len: 3, cap: 3
slice[0:4]  // паника
reflect.Value.Slice: slice index out of bounds
Как работает append для слайсов?
Можно ли применить к массивам?
Напиши свою функцию append.
slice := make([]int, 0, 3) 	// len: 0, cap: 3
slice
[]
slice = append(slice, 1)
slice = append(slice, 2, 3) 
slice
[1 2 3 1 2 3 1 2 3]
func main() {
	fmt.Println(Append([]int{1, 2, 3}, 4))
}

func Append(slice []int, elements ...int) []int {
    length := len(slice)
    capacity := length + len(elements)
    if capacity > cap(slice) {
        newSlice := make([]int, length, 2*capacity)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[:capacity]
    copy(slice[length:], elements)
    return slice
}



main()
[1 2 3 4]
Как можно добавить элементы в слайс?
Что будет если элемент не вмещается
в размер слайса?
// 1 способ - через append()

slice := make([]int, 0, 10) // len: 0, cap: 10
for i := 0; i < 10; i++ {
	slice = append(slice, i*2)
}

slice
[0 2 4 6 8 10 12 14 16 18]
// 2 способ - через индексы

slice := make([]int, 10) // len: 10, cap: 10
for i := 0; i < 10; i++ {
	slice[i] = i*2
}

slice
[0 2 4 6 8 10 12 14 16 18]
// у 2 способа есть проблемки - легко сделать переполнение 

slice := make([]int, 10) // len: 10, cap: 10
for i := 0; i < 11; i++ {
	slice[i] = i * 2
}

slice
reflect: slice index out of range
Как можно скопировать слайс?
Что такое функция copy?
Как добиться аналогичного поведения
copy с помощью append?
package main

import "fmt"

func main() {
    source := []int{1, 2, 3, 4, 5}

    target := make([]int, len(source)) // срез, куда копируем

    copy(target, source)

    fmt.Println("Скопированные элементы в target: ", target)
}

main()
Скопированные элементы в target:  [1 2 3 4 5]
package main

import "fmt"

func main() {
    slice1 := []int{1, 2, 3, 4, 5}

    slice2 := []int{}  // новый пустой срез

    // добавляем элементы из slice1 в slice2 по очереди
    for _, value := range slice1 {
        slice2 = append(slice2, value)
    }

    fmt.Println(slice2)
}

main()
[1 2 3 4 5]
Как можно нарезать слайс?
Какие есть подводные камни?
slice := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
subSlice := slice[3:5]

subSlice
[4 5]
subSlice[0] = 101
fmt.Println(subSlice)

fmt.Println(slice)
[101 5]
[1 2 3 101 5 6 7 8 9 10]
25 <nil>
slice := make([]int, 10, 25)
subSlice := slice[3:5]
fmt.Println(subSlice)

fmt.Println(len(slice), cap(slice))
fmt.Println(len(subSlice), cap(subSlice))

subSlice = append(subSlice, 11)
fmt.Println(subSlice)

fmt.Println(slice)
[0 0]
10 25
2 22
[0 0 11]
[0 0 0 0 0 11 0 0 0 0]
23 <nil>
Что такое table-driven тесты
и как их реализовать в Go?
package main

import (
	"fmt"
	"testing"
)

func MyFunction(num int) int {
	return num * 2
}

func TestMyFunction(t *testing.T) {
	cases := []struct {
		name  string
		input int
		want  int
	}{
		{"case1", 1, 2},
		{"case2", 2, 4},
		// ...
	}

	for _, tc := range cases {
		got := MyFunction(tc.input)
		if got != tc.want {
			t.Errorf("%s: got %d, want %d", tc.name, got, tc.want)
		}
	}
}

func main() {
	testing.Main(func(pat, str string) (bool, error) { return true, nil }, []testing.InternalTest{
		{"TestMyFunction", TestMyFunction},
	})
	fmt.Println("All tests passed!")
}
repl.go:33:22: not enough arguments in call to testing.Main:
	have (func(string, string) (bool, error), []testing.InternalTest)
	want (func(string, string) (bool, error), []testing.InternalTest, []testing.InternalBenchmark, []testing.InternalExample)
В каких случаях в Go могут возникнуть
deadlocks?
package main

import "fmt"

func main() {
	ch := make(chan string, 4)  // канал ёмкости 4

    // происходит 4 записи
	ch <- "hello"
	ch <- "I"
	ch <- "am"
	ch <- "David"
	ch <- "David"

    // ...и 4 чтения
	fmt.Println(<-ch)
	fmt.Println(<-ch)
	fmt.Println(<-ch)
	fmt.Println(<-ch)
}

main()
hello true
I true
am true
David true
package main

import "fmt"

func main() {
	ch := make(chan string, 4)  // канал ёмкости 4

    // происходит 4 записи
	ch <- "hello"
	ch <- "I"
	ch <- "am"
	ch <- "David"
    // ch <- "Hmmm"

    // ...и 4 чтения
	fmt.Println(<-ch)
	fmt.Println(<-ch)
	fmt.Println(<-ch)
	fmt.Println(<-ch)
}

main()
hello true
I true
am true
David true
Что такое горутина? Как ее остановить?
import "time"

func printNumbers() {
    for i := 1; i <= 5; i++ {
        fmt.Println(i)
    }
}

func main() {
    go printNumbers()
    time.Sleep(1 * time.Second)  // ожидаем завершения горутины
}

main()
1
2
3
4
5
package main

import (
    "fmt"
)

func printNumbers(ch chan int) {
    for i := 1; i <= 5; i++ {
        ch <- i  // отправляем значение в канал
    }
    close(ch)  // закрываем канал после передачи всех значений
}

func main() {
    ch := make(chan int)  // создаём канал
    go printNumbers(ch)
    for num := range ch {
        fmt.Println(num)
 }
}

main()
1
2
3
4
5
Как завершить много горутин?
package main

import (
   "fmt"
   "sync"
   "time"
)

func main() {
   fmt.Println("Начало функции main()")
   var wg sync.WaitGroup
   for i := 1; i < 4; i++ {
      wg.Add(1)          // Увеличиваем счетчик потоков на единицу
      go func(n int) {
         defer wg.Done() // Уменьшаем счетчик потоков на единицу
         for j := 1; j < 11; j++ {
            fmt.Println("Поток:", n, "j =", j)
            time.Sleep(time.Second) // Имитация выполнения задачи
         }
      }(i)
   }
   wg.Wait() // Ожидаем завершения всех потоков
   fmt.Println("Конец функции main()")
}
Converter.Type(): unsupported types.Type: *types.TypeParam
// примерно так выглядит вывод в консоль
Начало функции main()
Поток: 1 j = 1
Поток: 3 j = 1
Поток: 2 j = 1
....
Поток: 1 j = 8
Поток: 1 j = 9
Поток: 3 j = 9
Поток: 2 j = 9
Поток: 2 j = 10
Поток: 3 j = 10
Поток: 1 j = 10
Конец функции main()
Реализовать функцию reverse,
разворачивающую срез целых чисел
без использования временного среза
package main

import "fmt"

func reverse(sw []int) {
  for a, b := 0, len(sw)-1; a < b; a, b = a+1, b-1 {
    sw[a], sw[b] = sw[b], sw[a]
  } 
}

func main() { 
  x := []int{3, 2, 1, 0} 
  reverse(x)
  fmt.Println(x)
}


main()
[0 1 2 3]
Что такое глобальная переменная?
package main

import "fmt"

var x int = 10                    // Глобальная переменная

func main() {
   test()
   // Вывод значения глобальной переменной x
   fmt.Println(x)                 // 10
   { // Блок
      z := 30                     // Локальная переменная
      fmt.Println(z)              // 30
   }
   // Переменная z здесь уже не видна!!!
   for i := 0; i < 5; i++ {
      fmt.Println(i)              // 0, 1, 2, 3, 4
   }
   // Переменная i здесь уже не видна!!!
}
func test() {
   var x int = 5                  // Локальная переменная
   // Вывод значения локальной переменной x
   fmt.Println(x)                 // 5
}


main()
5
10
30
0
1
2
3
4
Напиши алгоритм бинарного поиска
package main

func BinarySearch(in []int, searchFor int) (int, bool) {
  if len(in) == 0 {
    return 0, false
  }

  var first, last = 0, len(in) - 1

  for first <= last {
    var mid = ((last - first) / 2) + first

    if in[mid] == searchFor {
      return mid, true
    } else if in[mid] > searchFor { // нужно искать в "левой" части слайса
      last = mid - 1
    } else if in[mid] < searchFor { // нужно искать в "правой" части слайса
      first = mid + 1
    }
  }

  return 0, false
}

BinarySearch([]int{4, 60, 79, 91}, 79)
2 true
Что выведет этот код?
package main

import (
	"fmt"
)

func main() {
	test1 := []int{1, 2, 3, 4, 5}
	test1 = test1[:3]
	test2 := test1[3:]
	fmt.Println(test2[:2])
}

main()
[4 5]
Что ты можешь сказать про
структуру Reader?
package main

import "os"

func main() {
    buf := bytes.NewReader([]byte("test"))
    buf.WriteTo(os.Stdout) // test

    arr := []byte{0, 0}
    buf := bytes.NewReader([]byte("test"))
    fmt.Println(buf.Read(arr)) // 2 <nil>
    fmt.Println(arr)           // [116 101]
    fmt.Println(buf.Read(arr)) // 2 <nil>
    fmt.Println(arr)           // [115 116]
    fmt.Println(buf.Read(arr)) // 0 EOF
}

main()
repl.go:6:12: undefined "bytes" in bytes.NewReader <*ast.SelectorExpr>
Как реализована map в Go?
first_map := map[int]bool {0: true, 1: false, 2:true}
first_map
map[0:true 1:false 2:true]
Что следует учитывать при добавлении
элемента в мапу во время итерации,
чтобы избежать
недетерминированных результатов?
m := map[int]bool {
    0: true,
    1: false,
    2: true, }

    for k, v := range m {
        if v {
            m[10+k] = true
        }
}

fmt.Println(m)
map[0:true 1:false 2:true 10:true 12:true 20:true 22:true]
59 <nil>
package main

import "fmt"

func copyMap(m map[int]bool) map[int]bool {
    m2 := make(map[int]bool)
    for k, v := range m {
        m2[k] = v
    }
    return m2
}

func main() {
    m := map[int]bool{
        0: true,
        1: false,
        2: true,
    }
    
    m2 := copyMap(m)

    for k, v := range m {
        m2[k] = v
        if v {
            m2[10+k] = true
        }
    }
    
    fmt.Println(m2)
}

main()
map[0:true 1:false 2:true 10:true 12:true]
Что важно помнить при использовании мапы
типа any?
func getMessage() {
    return "hello"
}

b := getMessage()
var m map[string]any
err := json.Unmarshal(b, &m)
if err != nil {
    return err 
}
repl.go:2:5: return: expecting 0 expressions, found 1: return "hello"
Что такое data race (гонка данных) в Go?
func main() {
    c := make(chan bool)
    m := make(map[string]string)
    go func() {
        m["1"] = "a" // Первый конфликтный доступ
        c <- true
    }()
    m["2"] = "b" // Второй конфликтный доступ
    <-c
    for k, v := range m {
        fmt.Println(k, v)
    }
}

main()
1 a
2 b
Вывести все комбинации
символов строки
package main
import "fmt"

// Perm вызывает f с каждой пермутацией a.
func Perm(a []rune, f func([]rune)) {
  perm(a, f, 0)
}

// пермутируем значения в индексе i на len(a)-1.
func perm(a []rune, f func([]rune), i int) {
  if i > len(a) {
    f(a)
    return
  }
  perm(a, f, i+1)
  for j := i + 1; j < len(a); j++ {
    a[i], a[j] = a[j], a[i]
    perm(a, f, i+1)
    a[i], a[j] = a[j], a[i]
  }
}

func main() {
  Perm([]rune("ab"), func(a []rune) {
    fmt.Println(string(a))
  })
}

main()  // go run название_файла.go
ab
ba
Что такое интерфейсы в Go?
package main

type animal interface {  // этот интерфейс реализует метод
	makeSound()          //                   makeSound()
}

type cat struct{}
type dog struct{}

func (c *cat) makeSound() {
	fmt.Println("meow!")
}

func (d *dog) makeSound() {
	fmt.Println("woof!")
}

func main() {
	var c, d animal = &cat{}, &dog{}
	c.makeSound()
	d.makeSound()
}

main()
meow!
woof!
// мы можем даже передавать интерфейс другой функции
package main

import "fmt"

type greeter interface {
	greet(string) string
}

type russian struct{}
type american struct{}

func (r *russian) greet(name string) string {
	return fmt.Sprintf("Привет, %s!", name)
}

func (r *american) greet(name string) string {
	return fmt.Sprintf("Hello, %s!", name)
}

func sayHello(g greeter, name string) {
	fmt.Println(g.greet(name))
}

func main() {
	sayHello(&russian{}, "Вася")
	sayHello(&american{}, "Lucy")
}

main()
Привет, Вася!
Hello, Lucy!
Как сообщить компилятору Go,
что наш тип реализует интерфейс?
type Speaker interface {
    Speak() string
}

type Person struct {
    Name string
}

func (p Person) Speak() string {
    return "My name is " + p.Name
}

func main() {
    p := Person()
    p.Speak()
}
Написать функцию,
находящую палиндром
// Вариант №1: Сравнение символов

package main 

import "fmt"

func main() {
    fmt.Println(IsPalindrome("olloj"))
}    

func IsPalindrome(str string) bool {
  for i := 0; i < len(str)/2; i++ {
    if str[i] != str[len(str)-i-1] {
      return false
    }
  }

  return true
}

main()
false
// Вариант №2: Использование функций strings

package main

import (
    "strings"
    "fmt"
        )

func main() {
    fmt.Println(IsPalindrome("acba"))
}    

func IsPalindrome(str string) bool {
  reversedStr := strings.Builder{}

  for i := len(str) - 1; i >= 0; i-- {
    reversedStr.WriteByte(str[i])
  }

  return str == reversedStr.String()
}

main()
false
// Вариант №3: Использование пакета bytes

package main

import (
    "bytes"
    "fmt"
        )

func main() {
    fmt.Println(IsPalindrome("abcba"))
}   

func IsPalindrome(str string) bool {
  reversedBytes := make([]byte, len(str))

  for i := 0; i < len(str); i++ {
    reversedBytes[i] = str[len(str)-i-1]
  }

  return bytes.Equal([]byte(str), reversedBytes)
}

main()
true
// Вариант №4: Рекурсия

package main

import "fmt"

func main() {
    fmt.Println(IsPalindrome("abcba"))
}   

func IsPalindrome(str string) bool {
  if len(str) <= 1 {
    return true
  }

  if str[0] != str[len(str)-1] {
    return false
  }

  return IsPalindrome(str[1 : len(str)-1])
}

main()
true
