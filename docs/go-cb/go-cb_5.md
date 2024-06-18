# 第六章：数据结构实例

# 6.0 简介

Go 语言有 4 种基本的数据结构 — 数组、切片、映射和结构体。我们单独有一章讨论结构体，所以我们将分开讨论它。在本章中，我们只讨论数组、切片和映射。我们将先讲解它们的背景信息，然后再深入讨论它们的具体用法。

## 数组

数组是表示相同类型元素的有序序列的数据结构。数组的大小是静态的，在定义数组时设置，之后无法更改。数组是值类型。这是一个重要的区别，因为在某些语言中，数组类似于指向数组中第一个项的指针。这意味着如果我们将数组传递给函数，我们将传递数组的副本，这可能会很昂贵。

## 切片

切片也是表示有序元素序列的数据结构。事实上，切片是建立在数组之上的，并且由于其灵活性，比数组更常用。切片没有固定的长度。在内部，切片是一个结构体，包含一个指向数组的指针，数组段的长度以及底层数组的容量。

## 映射

映射是一种将一个类型的值（称为*键*）与另一个类型的值（称为*值*）相关联的数据结构。这样的数据结构在许多其他编程语言中很常见，有不同的名称，如哈希表、哈希映射和字典。在内部，映射是一个指向`runtime.hmap`结构的指针。

理解这 3 种数据结构很重要，它们是所有其他数据结构的基础构建模块，但它们彼此之间也有根本性的不同。简而言之，数组是固定长度的有序列表，是值类型。切片是一个结构体，其第一个元素是指向数组的指针。映射是指向内部哈希映射结构的指针。

# 6.1 创建数组或切片

## 问题

你想要创建数组或切片。

## 解决方案

创建数组或切片的方法有很多，包括直接使用文字、从另一个数组创建，或使用`make`函数。

## 讨论

数组和切片在概念上有很大的差异，但在概念上它们非常相似。因此，创建数组和切片也非常相似。

### 定义数组

你可以通过在方括号中声明数组的大小，然后跟着元素的数据类型来定义数组。数组和切片只能包含相同类型的元素。你也可以在声明时用花括号初始化数组。

```go
var numbers [10]int
fmt.Println(numbers)
rhyme := [4]string{"twinkle", "twinkle", "little", "star"}
fmt.Println(rhyme)
```

如果你运行上面的代码片段，你将会看到这个结果。

```go
[0 0 0 0 0 0 0 0 0 0]
[twinkle twinkle little star]
```

int 或 float 数组的默认值为 0。请注意，一旦创建数组，数组的大小就不能再改变，但元素可以改变。

### 定义切片

切片是建立在数组之上的构造。大多数情况下，当需要处理有序列表时，通常会使用切片，因为它们更灵活，而且如果底层数组很大，使用起来也更便宜。

切片的定义方式完全相同，只是不提供切片的大小。

```go
var integers []int
fmt.Println(integers)
var sheep = []string{"baa", "baa", "black", "sheep"}
fmt.Println(sheep)
```

如果你运行上述代码片段，你将看到这个。

```go
[]
[baa baa black sheep]
```

我们还可以通过`make`函数创建切片。

```go
var integers = make([]int, 10)
fmt.Println(integers)
```

如果使用`make`，需要提供类型、长度和可选的容量。如果不提供容量，默认为给定的长度。如果你运行上述片段，你将看到这个。

```go
[0 0 0 0 0 0 0 0 0 0]
```

正如你所看到的，`make`也初始化了切片。

要找出数组或切片的长度，可以使用`len`函数。要找出数组或切片的容量，可以使用`cap`函数。

```go
integers = make([]int, 10, 15)
fmt.Println(integers)
fmt.Println("length:", len(integers))
fmt.Println("capacity:", cap(integers))
```

上面的`make`函数分配了一个包含 15 个整数的数组，然后创建了一个长度为 10、容量为 15 的切片，指向数组的前 10 个元素。

如果你运行上述代码，这就是你会得到的结果。

```go
[0 0 0 0 0 0 0 0 0 0]
length: 10
capacity: 15
```

我们还可以用`new`方法创建新的切片。

```go
var ints *[]int = new([]int)
fmt.Println(ints)
```

`new`方法不直接返回切片，它只返回一个指向切片的指针。它也不初始化切片，而是将其置零。运行上述代码时，你会看到得到什么。

```go
&[]
```

我们不能使用`make`函数创建新数组，但可以使用`new`函数创建新数组。

```go
var ints *[10]int = new([10]int)
fmt.Println(ints)
```

我们得到的是一个指向数组的指针如下。

```go
&[0 0 0 0 0 0 0 0 0 0]
```

# 6.2 访问数组或切片

## 问题

你想访问数组或切片中的元素。

## 解决方案

有几种方法可以访问数组或切片中的元素。数组和切片是有序列表，因此可以通过它们的索引访问元素。可以通过单个索引或索引范围访问元素。还可以通过迭代元素访问它们。

## 讨论

访问数组和切片几乎是相同的。由于它们是有序列表，我们可以通过索引访问数组或切片的元素。

```go
numbers := []int{3, 14, 159, 26, 53, 59}
```

给定上面的切片，给定索引 3（从 0 开始），切片中的第 4 个元素是 25，并可以使用变量名，后跟方括号和方括号内的索引访问。

```go
numbers[3] // 26
```

我们还可以通过使用起始索引，后跟冒号`:`和结束索引来访问一系列数字。结束索引不包括在内，这导致一个切片（当然）。

```go
numbers[2:4] // [159, 26]
```

如果没有起始索引，切片将从 0 开始。

```go
numbers[:4] // [3 14 159 26]
```

如果没有结束索引，切片将以原始切片（或数组）的最后一个元素结束。

```go
numbers[2:] // [159 265 53 59]
```

毫无疑问，如果你没有起始索引或结束索引，将返回整个原始切片。虽然这听起来很愚蠢，但这确实有其有效用途——它简单地将一个数组转换为切片。

```go
numbers := [6]int{3, 14, 159, 26, 53, 59} // an array
numbers[:] // this is a slice
```

我们还可以通过迭代数组或切片来访问数组或切片中的元素。

```go
for i := 0; i < len(numbers); i++ {
    fmt.Println(numbers[i])
}
```

这使用了一个普通的`for`循环，迭代切片的长度，每次循环增加计数。得到的输出如下。

```go
3
14
159
25
53
59
```

这使用了一个`for …​ range`循环，并返回索引`i`和值`v`。

```go
for i, v := range numbers {
    fmt.Printf("i: %d, v: %d\n", i, v)
}
```

得到的输出如下

```go
i: 0, v: 3
i: 1, v: 14
i: 2, v: 159
i: 3, v: 25
i: 4, v: 53
i: 5, v: 59
```

# 6.3 修改数组或切片

## 问题

您想要在数组或切片中添加、插入或删除元素。

## 解决方案

有几种方法可以修改数组或切片中的元素。元素可以附加到切片的末尾，插入到特定索引位置，删除或修改。

## 讨论

除了访问数组或切片中的元素外，您还可能希望在切片中添加、修改或删除元素。虽然无法在数组中添加或删除元素，但您始终可以修改其元素。

```go
numbers := []int{3, 14, 159, 26, 53, 58}
numbers[2] = 1000
```

当您修改给定索引处的元素时，它将相应地更改数组或切片。在这种情况下，当您运行代码时，将得到这个结果。

```go
[3 14 1000 26 53 58 97]
```

### 附加

数组无法改变其大小，因此无法向数组附加或添加元素。但是向切片附加元素非常简单。我们只需使用`append`函数，将其传递给切片和新元素，我们将得到一个具有附加元素的新切片。

```go
numbers := []int{3, 14, 159, 26, 53, 58}
numbers = append(numbers, 97)
fmt.Println(numbers)
```

如果您运行上述代码，您将得到这个结果。

```go
[3 14 159 26 53 58 97]
```

您不能将不同类型的元素附加到切片中。但是您可以向切片附加多个项目。

```go
numbers = append(numbers, 97, 932, 38, 4, 626)
```

这意味着您实际上可以使用切片解包符号`…​`将一个切片（或数组）附加到另一个切片中。

```go
nums := []int{97, 932, 38, 4, 626}
numbers = append(numbers, nums...)
```

但是同时附加一个元素和一个解包的切片是不允许的。您只能选择附加多个元素或一个解包的切片，但不能同时进行。

```go
numbers = append(numbers, 1, nums...) // this will produce an error
```

### 插入

虽然附加会向切片的末尾添加一个元素，但插入意味着在切片中的元素之间的任何位置添加一个元素。再次强调，这仅适用于切片，因为数组的大小是固定的。

与`append`不同，插入没有内置函数，但我们仍然可以使用`append`来完成任务。假设我们想在索引 2 和 3 之间的元素之间插入数字 1000。

```go
numbers := []int{3, 14, 159, 26, 53, 58}
numbers = append(numbers[:2+1], numbers[2:]...)
numbers[2] = 1000
```

首先，我们需要从原始切片的开头创建一个切片，到索引 2 加 1。这将为我们希望添加的新元素保留空间。接下来，我们将这个切片附加到另一个从索引 2 到原始切片末尾的切片中，使用解包符号。通过这种方式，我们在索引 2 和 3 之间创建了一个新元素。最后，我们在索引 2 处设置新元素 1000。

结果我们将得到这个新的切片。

```go
[3 14 1000 159 26 53 58]
```

如果我们想要在切片的开头添加一个元素怎么办？假设我们想要将整数 2000 添加到切片的开头。这很简单，我们只需将值以切片的形式附加到原始切片的解包值中。

```go
numbers = append([]int{2000}, numbers...)
```

这是在插入单个元素时的情况。如果我们想在另一个切片中间插入另一个切片怎么办？假设我们想在我们的`numbers`切片中间插入切片`[]int{1000, 2000, 3000, 4000}`。

有几种方法可以做到这一点，但我们将坚持使用`append`，这是最简洁的方法之一。

```go
numbers = []int{3, 14, 159, 26, 53, 58}
inserted := []int{1000, 2000, 3000, 4000}

tail := append([]int{}, numbers[2:]...)
numbers = append(numbers[:2], inserted...)
numbers = append(numbers, tail...)

fmt.Println(numbers)
```

首先，我们需要创建另一个切片，`tail`，来存储原始切片的*尾*部分。我们不能简单地对其进行切片并存储到另一个变量中（这称为*浅复制*），因为请记住——切片不是数组，它们是数组的一部分和其长度的指针。如果我们对`numbers`进行切片并存储到`tail`中，当我们改变`numbers`时，`tail`也会改变，这不是我们想要的。相反，我们希望通过将其附加到一个空的`int`切片来创建一个新的切片。

现在我们已经将`tail`放在一边，我们将`numbers`的头部附加到未打包的`inserted`中。最后，我们附加`numbers`（现在由原始切片的头部和`inserted`组成）和`tail`。这是我们应该得到的结果。

```go
[3 14 1000 2000 3000 4000 159 26 53 58]
```

### 删除

从切片中删除元素非常容易。如果它位于切片的开头或结尾，您只需相应地重新切片即可删除切片的开头或结尾。

让我们先取出切片的第一个元素。

```go
numbers := []int{3, 14, 159, 26, 53, 58}
numbers = numbers[1:] // remove element 0
fmt.Println(numbers)
```

当您运行上述代码时，您将得到这个结果。

```go
[14 159 26 53 58]
```

现在让我们先取出切片的最后一个元素。

```go
numbers := []int{3, 14, 159, 26, 53, 58}
numbers = numbers[:len(numbers)-1] // remove last element
fmt.Println(numbers)
```

当您运行上述代码时，您将得到这个结果。

```go
[3 14 159 26 53]
```

中间删除元素也很简单。您只需将原始切片的头部与原始切片的尾部附加在一起，移除其中的任何内容。在这种情况下，我们想要删除索引为 2 的元素。

```go
numbers := []int{3, 14, 159, 26, 53, 58}
numbers = append(numbers[:2], numbers[3:]...)
fmt.Println(numbers)
```

当我们运行上述代码时，我们得到了这个结果。

```go
[3 14 26 53 58]
```

# 6.4 使数组和切片在并发使用时安全

## 问题

您希望通过多个 goroutine 安全地使用数组和切片。

## 解决方案

使用`sync`库中的互斥体（mutex）来保护数组或切片。在修改数组或切片之前对其进行锁定，并在修改完成后解锁。

## 讨论

数组和切片在并发使用时不安全。如果要在多个 goroutine 之间共享一个切片或数组，则需要使其免受竞争条件的影响。Go 语言提供了一个`sync`包，特别是`Mutex`。

首先看一下竞争条件是如何产生的。竞争条件发生在多个 goroutine 试图同时访问共享资源时。

```go
var shared []int = []int{1, 2, 3, 4, 5, 6}

// increase each element by 1
func increase(num int) {
	fmt.Printf("[+%d a] : %v\n", num, shared)
	for i := 0; i < len(shared); i++ {
		time.Sleep(20 * time.Microsecond)
		shared[i] = shared[i] + 1
	}
	fmt.Printf("[+%d b] : %v\n", num, shared)
}

// decrease each element by 1
func decrease(num int) {
	fmt.Printf("[-%d a] : %v\n", num, shared)
	for i := 0; i < len(shared); i++ {
		time.Sleep(10 * time.Microsecond)
		shared[i] = shared[i] - 1
	}
	fmt.Printf("[-%d b] : %v\n", num, shared)
}
```

在上面的示例中，我们有一个名为`shared`的整数切片，被两个名为`increase`和`decrease`的函数使用。这两个函数简单地逐个获取共享切片中的元素并分别增加或减少 1。但在增加或减少元素之前，我们等待了一个非常短的时间，其中`increase`函数等待的时间更长。这模拟了多个 goroutine 之间时间差异的情况。我们在修改共享元素之前打印出`shared`切片的状态，并在修改后再次打印出来以展示其状态变化。

我们从`+main`调用`increase`和`decrease`函数，并且每次函数调用都作为一个单独的 goroutine。程序结束时，我们稍等片刻以确保所有的 goroutine 都完成（否则所有的 goroutine 将在程序结束时结束）。

```go
func main() {
	for i := 0; i < 5; i++ {
		go increase(i)
	}
	for i := 0; i < 5; i++ {
		go decrease(i)
	}
	time.Sleep(2 * time.Second)
}
```

当我们运行程序时，你会看到类似这样的输出。

```go
[-4 a] : [1 2 3 4 5 6]
[-1 a] : [0 2 3 4 5 6]
[-2 a] : [0 1 3 4 5 6]
[-3 a] : [0 1 2 4 5 6]
[+0 a] : [-2 1 2 3 5 6]
[+1 a] : [-3 -1 2 3 4 6]
[-4 b] : [-2 -2 1 3 4 5]
[+3 a] : [-2 -2 0 3 4 5]
[+4 a] : [-1 -1 -1 1 4 5]
[-1 b] : [1 0 0 0 1 4]
[-2 b] : [1 0 0 0 1 3]
[-3 b] : [1 0 0 0 1 2]
[+2 a] : [1 0 0 0 1 2]
[-0 a] : [2 2 1 1 1 2]
[+0 b] : [1 2 3 2 1 3]
[-0 b] : [1 2 3 3 2 2]
[+1 b] : [1 2 3 4 4 3]
[+3 b] : [1 2 3 4 4 4]
[+4 b] : [1 2 3 4 4 5]
[+2 b] : [1 2 3 4 5 6]
```

如果多次运行它，每次的结果可能会略有不同。你会注意到即使我们按顺序启动 goroutine（每次将顺序号发送给`modify`），实际执行的顺序是随机的，这是预期的行为。但我们不希望看到的是 goroutine 彼此重叠，共享切片根据哪个 goroutine 先访问而递增或递减。

例如，如果我们查看输出的第一行`[-4 a] : [1 2 3 4 5 6]`，会发现在调用减少每个元素的循环之前打印出来了。然后，在循环之后打印的行是`[-4 b] : [-2 -2 1 3 4 5]`，可以看到前 3 个元素并不符合预期！

同样，你会意识到即使在增加或减少元素的循环内部，重叠也会发生。

如何防止这种竞争条件？Go 语言在标准库中提供了`sync`包，它为我们提供了*mutex*或互斥锁。

```go
var shared []int = []int{1, 2, 3, 4, 5, 6}
var mutex sync.Mutex

// increase each element by 1
func increaseWithMutex(num int) {
	mutex.Lock()
	fmt.Printf("[+%d a] : %v\n", num, shared)
	for i := 0; i < len(shared); i++ {
		time.Sleep(20 * time.Microsecond)
		shared[i] = shared[i] + 1
	}
	fmt.Printf("[+%d b] : %v\n", num, shared)
	mutex.Unlock()
}

// decrease each element by 1
func decreaseWithMutex(num int) {
	mutex.Lock()
	fmt.Printf("[-%d a] : %v\n", num, shared)
	for i := 0; i < len(shared); i++ {
		time.Sleep(10 * time.Microsecond)
		shared[i] = shared[i] - 1
	}
	fmt.Printf("[-%d b] : %v\n", num, shared)
	mutex.Unlock()
}
}
```

```go
Using it is quite simple. Firstly we need to declare a mutex. Then, we call +Lock+ on the mutex before we start modifying the shared slice. This will lock up the shared slice such that nothing else can use it. When we're done, we call +Unlock+ to unlock the mutex.
```

如果像以前一样从`main`中调用这些函数，这里是输出结果。

```go
[-4 a] : [1 2 3 4 5 6]
[-4 b] : [0 1 2 3 4 5]
[+0 a] : [0 1 2 3 4 5]
[+0 b] : [1 2 3 4 5 6]
[+1 a] : [1 2 3 4 5 6]
[+1 b] : [2 3 4 5 6 7]
[+2 a] : [2 3 4 5 6 7]
[+2 b] : [3 4 5 6 7 8]
[+3 a] : [3 4 5 6 7 8]
[+3 b] : [4 5 6 7 8 9]
[+4 a] : [4 5 6 7 8 9]
[+4 b] : [5 6 7 8 9 10]
[-0 a] : [5 6 7 8 9 10]
[-0 b] : [4 5 6 7 8 9]
[-1 a] : [4 5 6 7 8 9]
[-1 b] : [3 4 5 6 7 8]
[-2 a] : [3 4 5 6 7 8]
[-2 b] : [2 3 4 5 6 7]
[-3 a] : [2 3 4 5 6 7]
[-3 b] : [1 2 3 4 5 6]
```

结果更加有条理。goroutine 不再重叠，元素的增加和减少有序而一致。

# 6.5 对切片数组进行排序

## 问题

你想要对数组或切片中的元素进行排序。

## 解决方案

对于`int`、`float64`和`string`数组或切片，你可以使用`sort.Ints`、`sort.Float64s`和`sort.Strings`。你也可以通过使用`sort.Slice`来使用自定义比较器。对于结构体，你可以通过实现`sort.Interface`接口来创建一个可排序的接口，然后使用`sort.Sort`来对数组或切片进行排序。

## 讨论

数组和切片是有序元素序列。然而，这并不意味着它们以任何方式排序，只是表示元素始终以相同的顺序排列。要对数组或切片进行排序，我们可以使用`sort`包中的各种函数。

对于`int`、`float64`和`string`，我们可以使用相应的`sort.Ints`、`sort.Float64s`和`sort.Strings`函数。

```go
integers := []int{3, 14, 159, 26, 53}
floats := []float64{3.14, 1.41, 1.73, 2.72, 4.53}
strings := []string{"the", "quick", "brown", "fox", "jumped"}

sort.Ints(integers)
sort.Float64s(floats)
sort.Strings(strings)

fmt.Println(integers)
fmt.Println(floats)
fmt.Println(strings)
```

如果我们运行上述代码，这就是我们将看到的。

```go
[3 14 26 53 159]
[1.41 1.73 2.72 3.14 4.53]
[brown fox jumped quick the]
```

这是按升序排序的。如果我们想要按降序排序怎么办？目前没有现成的函数可以按降序排序，但我们可以简单地使用一个 `for` 循环来反转排序后的切片。

```go
for i := len(integers)/2 - 1; i >= 0; i-- {
    opp := len(integers) - 1 - i
    integers[i], integers[opp] = integers[opp], integers[i]
}

fmt.Println(integers)
```

我们简单地找到切片的中间部分，然后使用循环，从中间开始，将元素与它们的对侧交换。如果我们运行上面的片段，你将会得到这样的结果。

```go
[159 53 26 14 3]
```

我们还可以使用 `sort.Slice` 函数，传入我们自己的 `less` 函数。

```go
sort.Slice(floats, func(i, j int) bool {
    return floats[i] > floats[j]
})
fmt.Println(floats)
```

这将产生输出。

```go
[4.53 3.14 2.72 1.73 1.41]
```

`less` 函数是 `sort.Slice` 函数的第二个参数，接受两个参数 `i` 和 `j`，是切片连续元素的索引。它的作用是在排序时，如果 `i` 处的元素小于 `j` 处的元素则返回 true。

如果元素相同怎么办？使用 `sort.Slice` 意味着元素的顺序可能与它们的原始顺序相反（或保持不变）。如果希望顺序始终与原始顺序一致，可以使用 `sort.SliceStable`。

`sort.Slice` 函数可以处理任何类型的切片，这意味着你也可以对自定义结构体进行排序。

```go
people := []Person{
	{"Alice", 22},
	{"Bob", 18},
	{"Charlie", 23},
	{"Dave", 27},
	{"Eve", 31},
}
sort.Slice(people, func(i, j int) bool {
	return people[i].Age < people[j].Age
})
fmt.Println(people)
```

如果你运行上述代码，你将会看到下面的输出，`people` 切片按照人们的年龄排序。

```go
[{Bob 18} {Alice 22} {Charlie 23} {Dave 27} {Eve 31}]
```

另一种排序结构体的方法是实现 `sort.Interface`。让我们看看如何为 `Person` 结构体做这个操作。

```go
type Person struct {
	Name string
	Age  int
}

type ByAge []Person

func (a ByAge) Len() int           { return len(a) }
func (a ByAge) Less(i, j int) bool { return a[i].Age < a[j].Age }
func (a ByAge) Swap(i, j int)      { a[i], a[j] = a[j], a[i] }
```

我们想要对结构体切片进行排序，所以我们需要将接口函数关联到切片，而不是结构体。我们创建一个名为 `ByAge` 的类型，它是 `Person` 结构体的切片。接下来，我们将 `Len`、`Less` 和 `Swap` 函数关联到 `ByAge`，使其成为一个实现了 `sort.Interface` 的结构体。这里的 `Less` 方法与我们在上面 `sort.Slice` 函数中使用的方法相同。

使用这个非常简单。我们将 `people` 强制转换为 `ByAge`，然后将其传入 `sort.Sort`。

```go
people := []Person{
	{"Alice", 22},
	{"Bob", 18},
	{"Charlie", 23},
	{"Dave", 27},
	{"Eve", 31},
}

sort.Sort(ByAge(people))
fmt.Println(people)
```

如果你运行上面的代码，你将会看到与下面相同的结果。

```go
[{Bob 18} {Alice 22} {Charlie 23} {Dave 27} {Eve 31}]
```

实现 `sort.Interface` 有点冗长，但显然有一些优势。首先，我们可以使用 `sort.Reverse` 按降序排序。

```go
sort.Sort(sort.Reverse(ByAge(people)))
fmt.Println(people)
```

这将产生以下输出。

```go
[{Eve 31} {Dave 27} {Charlie 23} {Alice 22} {Bob 18}]
```

你还可以使用 `sort.IsSorted` 函数来检查切片是否已经排序。

```go
sort.IsSorted(ByAge(people)) // true if it's sorted
```

最大的优势在于，使用 `sort.Interface` 比使用 `sort.Slice` 更高效。让我们做一个简单的基准测试。

```go
func BenchmarkSortSlice(b *testing.B) {
	f := func(i, j int) bool {
		return people[i].Age < people[j].Age
	}
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		sort.Slice(people, f)
	}
}

func BenchmarkSortInterface(b *testing.B) {
	for i := 0; i < b.N; i++ {
		sort.Sort(ByAge(people))
	}
}
```

这是基准测试的结果。

```go
$ go test -bench=BenchmarkSort
goos: darwin
goarch: arm64
pkg: github.com/sausheong/gocookbook/ch14_data_structures
BenchmarkSortSlice-10     	 9376766	       108.9 ns/op
BenchmarkSortInterface-10  	26790697	        44.33 ns/op
PASS
ok  	github.com/sausheong/gocookbook/ch14_data_structures	2.901s
```

正如你所见，使用 `sort.Interface` 更有效率。这是因为 `sort.Slice` 使用 `interface{}` 作为第一个参数。这意味着它可以接受任何结构体但效率较低。

# 6.6 创建地图

## 问题

你想创建新的地图。

## 解决方案

使用 `map` 关键字声明它，然后使用 `make` 函数进行初始化。在使用之前必须先初始化地图。

## 讨论

要创建地图，我们可以使用 `map` 关键字。

```go
var people map[string]int
```

上面的片段声明了一个名为`people`的映射，将一个字符串键映射到一个整数值。目前不能使用`people`映射，因为它的零值是 nil。要使用它，我们需要用`make`方法初始化它。

```go
people = make(map[string]int)
```

如果你觉得在声明和初始化中都要重复使用`map[string]int`看起来有些傻，你应该考虑同时做这两件事。

```go
people := make(map[string]int)
```

这将创建一个空映射。要填充映射，你可以将一个字符串映射到一个整数。

```go
people["Alice"] = 22
```

你也可以用这种方式初始化映射。

```go
people := map[string]int{
	"Alice":   22,
	"Bob":     18,
	"Charlie": 23,
	"Dave":    27,
	"Eve":     31,
}
```

如果你打印映射，它将如此显示。

```go
map[Alice:22 Bob:18 Charlie:23 Dave:27 Eve:31]
```

# 6.7 访问映射

## 问题

你想要访问映射中的键和值。

## 解决方案

使用方括号内的键来访问映射中的值。你也可以使用`for …​ range`循环来遍历映射。

## 讨论

访问给定键的值是直截了当的。只需使用方括号内的键来访问值。

```go
people := map[string]int{
	"Alice":   22,
	"Bob":     18,
	"Charlie": 23,
	"Dave":    27,
	"Eve":     31,
}

people["Alice"] // 22
```

如果键不存在会怎么样？什么都不会发生，Go 会简单地返回值类型的零值。在我们的例子中，整数的零值是 0，所以如果我们这样做。

```go
people["Nemo"] // 0
```

它将简单地返回 0。这可能不是我们要找的（特别是如果 0 是一个有效的响应），因此有一个机制可以检查键是否存在。

```go
age, ok := people["Nemo"]
if ok {
	// do whatever you want if the value exists
}
```

*逗号, ok*模式在许多情况下都很常见，也可以用来检查映射中是否存在键。使用方式很明显，如果键存在，`ok`就会变成 true，否则就是 false。尽管`ok`不是关键字，你可以使用任何变量名，它使用了多值赋值。尽管值仍然被返回，但由于你知道键不存在并且它只是一个零值，你可能不会使用它。

我们还可以使用`for …​ range`循环来遍历映射，就像我们处理数组和切片时所做的那样，不同之处在于，这里我们获取键和值。

```go
for k, v := range people {
	fmt.Println(k, v)
}
```

运行上面的代码将给我们这个输出。

```go
Alice 22
Bob 18
Charlie 23
Dave 27
Eve 31
```

如果只想要键，你可以省略从 range 中获取的第二个值。

```go
for k := range people {
	fmt.Println(k)
}
```

你将得到这个输出。

```go
Alice
Bob
Charlie
Dave
Eve
```

如果我们只想要值怎么办？没有特别的方法可以直接获取值，你必须使用相同的机制，并将它们放在一个切片中。

```go
var values []int
for _, v := range people {
	values = append(values, v)
}
fmt.Println(values)
```

你将得到这个输出。

```go
[22 18 23 27 31]
```

# 6.8 修改映射

## 问题

你想要修改或删除映射中的元素。

## 解决方案

使用`delete`函数从映射中删除键值对。要修改值，只需重新赋值。

## 讨论

修改值就是简单地覆盖现有的值。

```go
people["Alice"] = 23
```

`people["Alice"]`的值将变成 23。

要删除键，Go 提供了一个名为`delete`的内置函数。

```go
delete(people, "Alice")
fmt.Println(people)
```

这将是输出结果。

```go
map[Bob:18 Charlie:23 Dave:27 Eve:31]
```

如果尝试删除一个不存在的键会发生什么？什么都不会发生。

# 6.9 排序映射

## 问题

你想要按键排序映射。

## 解决方案

将映射的键获取到一个切片中并对该切片进行排序。然后使用排序后的键切片，再次遍历映射。

## 讨论

映射是无序的。这意味着每次遍历映射时，键值对的顺序可能与上次不同。那么我们如何确保每次都是相同的顺序呢？

首先，我们将键提取到一个切片中。

```go
var keys []string
for k := range people {
	keys = append(keys, k)
}
```

然后，我们根据需要对键进行排序。在这种情况下，我们要按照降序排序。

```go
// sort keys by descending order
for i := len(keys)/2 - 1; i >= 0; i-- {
	opp := len(keys) - 1 - i
	keys[i], keys[opp] = keys[opp], keys[i]
}
```

最后，我们可以按键的降序访问映射。

```go
for _, key := range keys {
	fmt.Println(key, people[key])
}
```

运行代码时，我们将看到这个结果。

```go
Eve 31
Dave 27
Charlie 23
Bob 18
Alice 22
```

# 作者简介

**张守祥**在软件开发行业已超过 27 年，并参与了多个行业的软件产品构建，使用了多种技术。他是 Java、Ruby 社区的活跃成员，现在主要专注于 Go 语言，在全球各地的会议上组织和发表演讲。他还主办着 GopherCon 新加坡，这是东南亚最大的社区主导的开发者大会之一，自 2017 年以来一直如此。张守祥已经写了 4 本编程书籍，其中 3 本是关于 Ruby，最后一本是关于 Go 的。
