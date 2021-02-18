
---
title: Golang 中 struct 嵌入 interface
date: 2021-02-18 18:20:00
tags: 术
---

最近越来越多会在一些开源项目和源码中见到 struct 嵌套匿名 interface，比如：

```go
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}

...

type reverse struct {
    Interface
}

```

本文试图搞清楚这种设计的意义以及我们在实际编码中何时会用上。

在这之前，首先需要搞清楚 interface 的一些小细节。

## Details of Interface

本文不对 interface 本身做科普，而是试图介绍一些 interface 中可能错过的小细节。

### 实际/动态类型和静态类型

```go
type Person struct {
	name string
}

func (p *Person) Jump() {
	fmt.Println("JUMP")
}

func (p *Person) Run() {
	fmt.Println("RUN")
}

func main() {
	var person Action = &Person{name: "Keroro"}
	person.Jump()
}
```

对于一个接口类型的变量来说，例如上面的变量  person，赋给它的值可以被叫做它的实际值（也称动态值），而该值的类型可以被叫做这个变量的实际类型（也称动态类型）。

比如，把取址表达式 &Person 的结果值赋给了变量 person，**值的类型 `*Person` 就是该变量的动态类型**。

动态类型这个叫法是相对于静态类型而言的。**对于变量 person 来讲，它的静态类型就是 Action，并且永远是 Action**，但是它的<u>动态类型却会随着赋给它的动态值而变化</u>。比如，只有我把一个 `*Person` 类型的值赋给变量 person 之后，该变量的动态类型才会是 `*Person` 。如果还有一个 Action 接口的实现类型 `*ET`，并且我又把一个此类型的值赋给了 `person`，那么它的动态类型就会变为 `*ET`。

**在给一个接口类型的变量赋予实际的值之前，它的动态类型是不存在的。**

### Struct with embemed Interface

一个实现了某个 interface 的 struct 需要实现该 interface 下的所有方法：

```go
type Action interface {
	Jump()
	Run()
}

type Person struct {
	name string
}

func (p *Person) Jump() {
	fmt.Println("JUMP")
}

// 编译失败：missing method Run
func main() {
  var p Action = &Person{"Keroro"}
}
```

但是如下声明是合法的：

```go
type Action interface {
	Jump()
	Run()
}

type Person struct {
	Action
}

func main() {
	var person Action = &Person{}
	// 打印结果：person:size=[16],value=[&{<nil>}]
	fmt.Printf("person:size=[%d],value=[%v]\n", unsafe.Sizeof(person), person)
}
```

Person 确实实现了Actiuon 的方法，从一个更可视化的角度来看，可以看成：

```go
func (p Person) Jump() {
  p.Action.Jump()
}
// Run 同理
```

那么 `p.Action.Jump()` 究竟指代的位置是什么？其实，**只要是任何实现了 Action 接口的任何对象都可以**。

另外需要注意的一点是上述例子的打印结果：`person:size=[16],value=[&{<nil>}]`，这里说明两点：

1. 一个 interface 占据的大小是 16 个字节;
2. 这里 person 的打印结果为 `&{<nil>}`，即 Action 接口的打印值为 nil，注意这里 nil 仅代表字面量上的 nil，而不代表此处 Action 的值为 nil，接口在被实例化时，会被包装一层复杂的数据结构 iface，iface 的实例会包含两个指针，**一个是指向类型信息的指针，另一个是指向动态值的指针**。这里的类型信息是由另一个专用数据结构的实例承载的，其中包含了动态值的类型，以及使它实现了接口的方法和调用它们的途径等等。因此，这里的 nil 的真正含义是指 Action 实例化后指向动态值的指针为 nil。

通过下面这个示例来更清楚的演示细节：

```go
type Action interface {
	Jump()
}

type Somebody struct {
	Action
}

func (p *Somebody) Jump() {
	fmt.Println("JUMP")
}

func main() {
	var somebody Somebody = Somebody{}
	var hero Somebody = Somebody{Action: &Somebody{}}
	somebody.Jump()	// 打印 JUMP，Somebody 结构实现了 Jump，与 Action 接口无关
	// runtime error: invalid memory address or nil pointer dereference
	// somebody.Action.Jump()
  hero.Action.Jump() // 打印 JUMP，Action 接口实例化，等价于 hero.Jump()

  // 打印结果：somebody:size=[16],value=[{<nil>}]
	fmt.Printf("somebody:size=[%d],value=[%v]\n", unsafe.Sizeof(somebody), somebody)
  // 打印结果：hero:size=[16],value=[{0xc000010200}]
	fmt.Printf("hero:size=[%d],value=[%v]\n", unsafe.Sizeof(hero), hero)
}

```

## Use Case

回到最开始的问题，什么时候会用上它们？

### Example: Interface Wrapper

从一个角度来说，它可以做到 DRY 原则，另一方面，它同样可以帮助开发者遵守 SOLID 中的依赖倒置原则。如下示例：

```go
type Action interface {
	Jump()
	Run()
}

type Somebody struct{}

func (p *Somebody) Jump() {
	fmt.Println("SOMEBODY JUMP")
}

func (p *Somebody) Run() {
	fmt.Println("SOMEBODY RUN")
}

type Hero struct {
	Action
}

func (p *Hero) Jump() {
	fmt.Println("HERO JUMP")
}

func main() {
	var hero = Hero{&Somebody{}}
	hero.Run()
	hero.Jump()
}

// 输出：
// SOMEBODY RUN
// HERO JUMP
```

与 Somebody 结构体相比，Hero 结构体只有一个 Jump 方法和 Somebody 结构体不同，那么它就可以采用这种做法。Hero 可以只实现 Jump 方法而不需要考虑 Run 方法的实现，相当于 Hero 这个数据结构 “继承” 了 Somebody 这个数据结构，同时它“实现”了自己的 Jump 方法。

### Example: sort.Reverse

一个经典的例子是 golang 源码库中的 `sort.Reverse`。它的使用在乍看之下并不好理解。一个比较简单的排序例子，对一个整数切片进行排序：

```go
lst := []int{4, 5, 2, 8, 1, 9, 3}
sort.Sort(sort.IntSlice(lst))
fmt.Println(lst)

//打印：[1 2 3 4 5 8 9]
```

`sort.Sort` 接受一个实现了 `sort.Interface` 接口的参数，定义如下：

```go
type Interface interface {
    // Len is the number of elements in the collection.
    Len() int
    // Less reports whether the element with
    // index i should sort before the element with index j.
    Less(i, j int) bool
    // Swap swaps the elements with indexes i and j.
    Swap(i, j int)
}
```

如果有一个类型想用 `sort.Sort` 来排序，就必须实现这个接口；对于像 int slice 这样的简单类型，标准库提供了 `sort.IntSlice` 这样的方便类型，可以接受传进来的切片，并在其上实现 `sort.Interface` 方法。

```go
// IntSlice attaches the methods of Interface to []int, sorting in increasing order.
type IntSlice []int

func (p IntSlice) Len() int           { return len(p) }
func (p IntSlice) Less(i, j int) bool { return p[i] < p[j] }
func (p IntSlice) Swap(i, j int)      { p[i], p[j] = p[j], p[i] }
```

到目前为止，一切都很好。

那么 `sort.Reverse` 是如何工作的呢？通过巧妙地采用一个嵌入结构中的接口。sort 包中有这样一个类型来帮助完成任务。

```go
type reverse struct {
  sort.Interface
}

func (r reverse) Less(i, j int) bool {
  return r.Interface.Less(j, i)
}
```

注意这里 Less 的实现，逆向通过嵌入的方式实现 `sort.Interface`（只要在初始化的时候使用实现了该 struct 的类型就可以了），它“截取”了该接口的一个方法 `Less`。然后，它将其委托给嵌入值的 `Less`，但将参数的顺序倒置。这个 `Less` 实际上是将元素进行反向比较，这将使排序工作反向进行。

`sort.Reverse` 函数简单来说就是：

```go
func Reverse(data sort.Interface) sort.Interface {
  return &reverse{data}
}
```

现在可以这样做：

```go
lst := []int{4, 5, 2, 8, 1, 9, 3}
sort.Sort(sort.Reverse(sort.IntSlice(lst)))
fmt.Println(lst)
// 打印：[9 8 5 4 3 2 1]
```

这里需要理解的关键点是，调用 `sort.Reverse` 本身并不对任何东西进行排序或反转。它可以被看作是一个**高阶函数**：它产生一个值，包装给它的接口并调整其函数实现。对 `sort.Sort` 的调用是排序发生的地方。

## Reference

* [Meaning of a struct with embedded anonymous interface?](https://stackoverflow.com/questions/24537443/meaning-of-a-struct-with-embedded-anonymous-interface)

* [Embedding in Go: Part 3 - interfaces in structs](https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs/)

