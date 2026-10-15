# Go 面试题

## 语言基础

### 编译方式

Go 是编译型语言，可以通过 `go build` 编译程序。

### for range 的注意点

- 遍历数组或切片时，`value` 是元素的副本；要修改元素本身或获取其地址，应通过索引操作原数组或切片。
- 使用 [Go 1.22 及以后的语言语义](https://go.dev/blog/loopvar-preview)时，由 `:=` 声明的循环变量每轮独立；使用 `=` 给已有变量赋值时，仍会复用该变量。

### 函数与参数传递

- Go 的函数参数只有值传递，传入指针时，复制的也是指针值。
- 函数可以赋值给变量。

### 闭包

闭包是函数与其引用的外部变量的组合。即使外层函数已经返回，闭包仍可访问和修改这些变量。

```go
package main

import "fmt"

func getNumber() func() int {
	i := 0
	return func() int {
		i++
		return i
	}
}

func main() {
	// 同一个闭包每次调用都会修改同一个 i。
	nextNumber := getNumber()
	fmt.Println(nextNumber()) // 1
	fmt.Println(nextNumber()) // 2
	fmt.Println(nextNumber()) // 3

	// 新闭包拥有独立的 i。
	nextNumber1 := getNumber()
	fmt.Println(nextNumber1()) // 1
	fmt.Println(nextNumber1()) // 2
}
```

### 组合与继承

Go 不支持传统的类继承，通常通过结构体组合和嵌入实现代码复用。

## 并发与同步

### 并发设计原则

> Do not communicate by sharing memory; instead, share memory by communicating.

通过通信来共享内存。底层同步可以使用锁和原子操作，较高层的 goroutine 协作可以通过 channel 组织。

### sync 包概览

- `sync.Mutex`：互斥锁，保证同一时间只有一个 goroutine 访问临界区。
- `sync.RWMutex`：读写锁，允许多个 goroutine 同时读，写时独占。
- `sync.WaitGroup`：等待一组 goroutine 执行完成。
- `sync.Once`：保证某段代码只执行一次，常用于初始化。
- `sync.Pool`：复用临时对象，减少内存分配和 GC 压力。
- `sync.Map`：并发安全的 map，适合读多写少等特定场景。
- `sync.Cond`：条件变量，用于等待条件满足或唤醒等待的 goroutine。

另有 `sync/atomic` 包，提供原子读写、加减、CAS 等操作。

### select

`select` 用于选择可执行的 channel 收发操作；若多个分支同时就绪，会从中[伪随机选择一个](https://go.dev/ref/spec#Select_statements)。

### channel 原理

![channel 原理](/Users/bytedance/vscode/backend-knowledge/面试准备/assets/image-20261004143331153.png)

### context

**作用**：在调用链中传递请求级信息、截止时间和取消信号。

**`cancel()` 的原理**：

- `context.WithCancel` 创建可取消的 `cancelCtx`，并建立取消信号的父子传播关系。
- 调用 `cancel()` 时，会关闭对应的 `Done()` channel，并向下传播取消信号。
- goroutine 不会被强制终止，需要主动监听 `ctx.Done()` 并退出。

## defer 与接口

### defer 的实现

![defer 原理](/Users/bytedance/vscode/backend-knowledge/面试准备/assets/image-20261004151536191.png)

实现方式由编译器决定，简化如下：

```text
                 defer
                   │
                   ▼
       能不能 Open-coded？
           /            \
         能              不能
         │                │
         ▼                ▼
   Open-coded       defer 是否可能重复注册？
                       /       \
                     是         否
                     │          │
                     ▼          ▼
                  堆 _defer   栈 _defer
```

**Open-coded defer**：编译器把延迟调用的逻辑直接插入函数返回路径，减少创建 `_defer` 记录和维护链表的开销。

### interface 原理

![Go 接口底层数据结构详解](</Users/bytedance/vscode/backend-knowledge/面试准备/assets/Go 接口底层数据结构详解.png>)

- 空接口 `interface{}` 底层可理解为 `eface`：保存具体类型信息和数据指针 `data`。
- 非空接口底层可理解为 `iface`：保存 `itab` 和 `data`，其中 `itab` 记录接口类型、具体类型和方法表。
- 调用接口方法时，通过 `itab` 找到对应方法，并结合 `data` 完成调用。

## 内存管理

### 逃逸分析

Go 编译器通过逃逸分析，判断变量能否安全地分配在栈上。

变量的生命周期可能超过当前栈帧，或编译器无法证明栈分配安全时，就可能逃逸到堆上。常见场景有**返回局部变量指针、闭包捕获、被 goroutine 引用**，是否逃逸取决于具体用法和编译器优化。

### 内存分配

![Go 内存分配架构图](</Users/bytedance/vscode/backend-knowledge/面试准备/assets/Go 内存分配架构图.png>)

堆内存分配路径（当前层不足时才向后申请）：

- **小对象**：goroutine → P 的 `mcache` → `mcentral` → `mheap` → 操作系统。
- **大对象**：goroutine → `mheap` → 操作系统。

P 指 GMP 中的 P；`mcache` 由各个 P 独享，`mcentral` 由所有 P 共享。

### 垃圾回收（GC）

Go 使用**并发标记清除**算法，以三色标记描述对象状态，通过**写屏障**保证并发标记的正确性。

- **标记整理的取舍**：Go 的 GC 不移动堆对象，省去搬迁和更新引用的开销；按大小分级分配有助于缓解碎片。
- **分代 GC 的取舍**：部分短期对象可分配在栈上，减轻堆的回收压力；分代机制还需权衡收益与维护成本。
- **写屏障与 STW**：写屏障防止并发修改引用时漏标存活对象，支持并发标记，减少 STW（暂停程序执行）时间。

相关说明见 [Go GC 指南](https://go.dev/doc/gc-guide)。

## GMP 调度模型

### 整体结构

![GMP 调度模型示意图一](/Users/bytedance/vscode/backend-knowledge/面试准备/assets/image-20261004184022398.png)

![GMP 调度模型示意图二](/Users/bytedance/vscode/backend-knowledge/面试准备/assets/image-20261004184049277.png)

![GMP 调度模型示意图三](/Users/bytedance/vscode/backend-knowledge/面试准备/assets/image-20261004184111721.png)

### 与内存分配、阻塞的关系

每个 P 拥有独立的 `mcache`，常见的小对象分配路径无需加锁，可以减少竞争。

channel 操作或 `sync.Mutex` 加锁需要阻塞等待时，运行时会挂起当前 G，M 可以继续执行其他可运行的 G。

### G（goroutine）结构

![G 结构](/Users/bytedance/vscode/backend-knowledge/面试准备/assets/image-20261004184648556.png)

### M（Machine）结构

![M 结构](/Users/bytedance/vscode/backend-knowledge/面试准备/assets/image-20261004184729891.png)

### P（Processor）结构

![P 结构](/Users/bytedance/vscode/backend-knowledge/面试准备/assets/image-20261004184757577.png)

### 全局调度器

![全局调度器结构](/Users/bytedance/vscode/backend-knowledge/面试准备/assets/image-20261004184832088.png)
