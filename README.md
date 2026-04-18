Hello world示例

package main

import "std/console"

func main() {
    console.println("Hello, Tea!")
}


示例系统内核，语言在设计阶段，暂时不可用


kernel.t

```tea
package kernel

// 直接操作 VGA 文本缓冲区（物理地址 0xB8000）
const VGA_MEM = 0xB8000 as *mut u16
const VGA_WIDTH = 80
const VGA_HEIGHT = 25

// 颜色：黑底灰字
const COLOR = 0x07

// 内核入口（链接器设置为 _start）
@export(name="_start", linkage="strong")
pub fn _start() {
    let msg = "Tea Kernel v0.1 - Manual memory, no GC"

    // 直接写入显存
    for i in 0..len(msg) {
        let offset = i * 2
        unsafe {
            let vga = VGA_MEM as *mut u8
            vga[offset] = msg[i]           // 字符
            vga[offset + 1] = COLOR        // 属性
        }
    }

    // 停机（hlt 指令）
    loop {
        unsafe { asm("hlt") }
    }
}
```

链接脚本片段 (linker.ld)：

```ld
ENTRY(_start)
SECTIONS {
    . = 0x100000;
    .text : { *(.text*) }
    .data : { *(.data*) }
    .bss  : { *(.bss*) }
}
```

编译：

```bash
tea build --target=kernel -o kernel.elf kernel.t
```

---

3. 变量与基本类型

variables.t

```tea
package main

import "std/console"

func main() {
    // 显式声明
    var age int = 30
    var name string = "Tea"

    // 类型推导
    pi := 3.14159
    isGood := true

    // 常量
    const MAX_SIZE = 1024

    // 多变量赋值
    x, y := 10, 20
    x, y = y, x  // 交换

    // 指针
    var ptr *int = &age
    *ptr = 31

    console.println("name:", name, "age:", age, "pi:", pi, "isGood:", isGood)
    console.println("x:", x, "y:", y)
    console.println("MAX_SIZE:", MAX_SIZE)
}
```

---

4. 函数

functions.t

```tea
package main

import "std/console"

// 基本函数
func add(a int, b int) int {
    return a + b
}

// 多返回值（类似 Go）
func divide(a, b int) (int, int) {
    return a / b, a % b
}

// 命名返回值
func sum(nums []int) (total int) {
    for _, n := range nums {
        total += n
    }
    return  // 裸返回，自动返回 total
}

// 可变参数
func printAll(msgs ...string) {
    for _, msg := range msgs {
        console.print(msg, " ")
    }
    console.println()
}

// 闭包
func counter() func() int {
    count := 0
    return func() int {
        count++
        return count
    }
}

func main() {
    console.println("add(3,5):", add(3, 5))

    q, r := divide(17, 5)
    console.println("17/5 =", q, "余", r)

    console.println("sum of [1,2,3,4] =", sum([]int{1, 2, 3, 4}))

    printAll("Tea", "is", "smooth")

    c := counter()
    console.println(c())  // 1
    console.println(c())  // 2
}
```

---

5. 结构体与方法

structs.t

```tea
package main

import "std/console"

type Person struct {
    name string
    age  int
}

// 值接收者
func (p Person) greet() {
    console.println("Hello, I'm", p.name)
}

// 指针接收者（可修改）
func (p *Person) birthday() {
    p.age++
}

// 构造函数（手动分配内存）
func NewPerson(name string, age int) *Person {
    p := alloc(Person)
    p.name = name
    p.age = age
    return p
}

func (p *Person) free() {
    free(p)
}

func main() {
    // 栈分配（自动管理，函数返回后无效）
    alice := Person{"Alice", 25}
    alice.greet()

    // 堆分配（需手动释放）
    bob := NewPerson("Bob", 30)
    defer bob.free()
    bob.birthday()
    console.println("Bob is now", bob.age)
}
```

---

6. 接口

interfaces.t

```tea
package main

import "std/console"

// 定义接口
type Speaker interface {
    speak() string
}

// 实现接口（隐式，类似 Go）
type Dog struct {
    name string
}

func (d Dog) speak() string {
    return "Woof! I'm " + d.name
}

type Cat struct {
    name string
}

func (c Cat) speak() string {
    return "Meow! I'm " + c.name
}

// 使用接口的函数
func announce(s Speaker) {
    console.println(s.speak())
}

func main() {
    dog := Dog{"Rex"}
    cat := Cat{"Luna"}

    announce(dog)
    announce(cat)

    // 类型断言
    var s Speaker = dog
    if d, ok := s.(Dog); ok {
        console.println("It's a dog named", d.name)
    }
}
```

---

7. 手动内存管理（核心特性）

manual_mem.t

```tea
package main

import "std/console"
import "std/mem"

type Node struct {
    value int
    next  *Node
}

// 创建链表（完全手动）
func createList(vals []int) *Node {
    var head *Node
    for i := len(vals)-1; i >= 0; i-- {
        node := alloc(Node)
        node.value = vals[i]
        node.next = head
        head = node
    }
    return head
}

// 释放链表
func freeList(head *Node) {
    for head != nil {
        next := head.next
        free(head)
        head = next
    }
}

// 区域分配（Arena）：批量分配，一次释放
func processWithArena() {
    arena := mem.new_arena()
    defer mem.free_arena(arena)

    // 从 arena 分配，无需逐个 free
    buf := mem.arena_alloc_array(arena, byte, 4096)
    // 使用 buf...
    console.println("Arena allocated buffer at", buf)
    // arena 会在 defer 中整体释放
}

func main() {
    // 示例1：链表
    list := createList([]int{10, 20, 30, 40})
    defer freeList(list)

    for p := list; p != nil; p = p.next {
        console.println(p.value)
    }

    // 示例2：区域分配
    processWithArena()
}
```

---
8. 并发（Goroutine + Channel）

concurrency.t

```tea
package main

import "std/console"
import "std/time"

func worker(id int, jobs <-chan int, results chan<- int) {
    for job := range jobs {
        console.println("worker", id, "processing job", job)
        time.sleep(100 * time.millisecond)
        results <- job * 2
    }
}

func main() {
    const numJobs = 5
    const numWorkers = 3

    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)

    // 启动 workers
    for w := 1; w <= numWorkers; w++ {
        go worker(w, jobs, results)
    }

    // 发送任务
    for j := 1; j <= numJobs; j++ {
        jobs <- j
    }
    close(jobs)

    // 收集结果
    for r := 1; r <= numJobs; r++ {
        res := <-results
        console.println("result:", res)
    }
}
```

---

9. 错误处理（多返回值模式）

errors.t

```tea
package main

import "std/console"
import "std/strconv"

// 定义错误类型（简单字符串）
type Error string
func (e Error) error() string { return string(e) }

const ErrDivideByZero = Error("division by zero")

func safeDivide(a, b int) (int, Error) {
    if b == 0 {
        return 0, ErrDivideByZero
    }
    return a / b, nil
}

func main() {
    // 类似 Go 的错误处理
    if res, err := safeDivide(10, 2); err != nil {
        console.println("error:", err)
    } else {
        console.println("10/2 =", res)
    }

    if _, err := safeDivide(10, 0); err != nil {
        console.println("expected error:", err)
    }

    // 使用 panic/recover（可选）
    defer func() {
        if r := recover(); r != nil {
            console.println("recovered from panic:", r)
        }
    }()
    panic("something went wrong")
}
```

---

10. 包与模块

math/math.t（自定义包）

```tea
package math

// 公开函数（首字母大写）
pub fn Add(a, b int) int {
    return a + b
}

pub fn Multiply(a, b int) int {
    return a * b
}
```

main.t（使用包）

```tea
package main

import "std/console"
import "myproject/math"   // 假设项目名为 myproject

func main() {
    sum := math.Add(3, 4)
    prod := math.Multiply(3, 4)
    console.println("3+4=", sum, "3*4=", prod)
}
```

---

编译与运行说明

· 普通程序：tea build <file.t> → 生成可执行文件。
· 内核：需 --target=kernel 并配合链接脚本，生成 ELF 文件后可用 QEMU 测试。
· 包管理：tea mod init <name> 创建项目；tea build 自动解析依赖。
