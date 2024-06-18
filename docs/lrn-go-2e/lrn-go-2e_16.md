# 第十六章。这里有龙：反射、不安全操作和 Cgo

已知世界的边缘是可怕的。古代的地图会在未探索的区域填上龙和狮子的图片。在前面的章节中，我强调过 Go 是一种安全的语言，有类型变量可以清楚地表明您使用的数据类型，有垃圾回收来管理内存。即使是指针也是温顺的；您无法像 C 和 C++ 那样滥用它们。

所有这些都是真实的，对于您将编写的大多数 Go 代码，您可以确信 Go 运行时会保护您。但是有逃逸通道。有时，您的 Go 程序需要涉足定义较少的领域。在本章中，您将了解如何处理无法通过普通 Go 代码解决的情况。例如，当编译时无法确定数据类型时，您可以使用 `reflect` 包中的反射支持来交互甚至构造数据。当您需要利用 Go 中数据类型的内存布局时，您可以使用 `unsafe` 包。如果只有使用 C 编写的库才能提供的功能，您可以使用 `cgo` 调用 C 代码。

您可能会想为什么这些高级概念会出现在一个针对 Go 初学者的书籍中。有两个原因。首先，寻找问题解决方案的开发人员有时会发现（并复制粘贴）他们不完全理解的技术。在将它们添加到代码库之前，最好了解一些可能会导致问题的高级技术。其次，这些工具很有趣。因为它们允许您使用 Go 通常不可能的功能，所以玩起来有点激动人心，看看您能做什么。

# 使用反射（reflection）允许您在运行时处理类型

使用 Go 的人喜欢它的一个原因是它是一种静态类型语言。大多数情况下，在 Go 中声明变量、类型和函数都很简单。当您需要一个类型、一个变量或一个函数时，您定义它：

```go
type Foo struct {
  A int
  B string
}

var x Foo

func DoSomething(f Foo) {
  fmt.Println(f.A, f.B)
}
```

您在编写程序时使用类型来表示您知道需要的数据结构。由于类型是 Go 的核心部分，编译器使用它们来确保您的代码正确。但有时，仅依赖编译时信息是有限制的。您可能需要使用在程序编写时不存在的信息，在运行时使用变量。也许您正在尝试将文件或网络请求中的数据映射到变量中。在这些情况下，您需要使用 *反射*。反射允许您在运行时检查类型。它还提供了在运行时检查、修改和创建变量、函数和结构体的能力。

这引出了这种功能何时需要的问题。如果您查看 Go 标准库，您可以得到一个想法。它的用途分为几个一般类别之一：

+   从数据库读写数据。`database/sql` 包使用反射将记录发送到数据库并读取数据。

+   Go 内置的模板库 `text/template` 和 `html/template` 使用反射来处理传递给模板的值。

+   `fmt` 包大量使用反射，因为所有对 `fmt.Println` 和其它友元函数的调用都依赖于反射来检测提供参数的类型。

+   `errors` 包使用反射来实现 `errors.Is` 和 `errors.As`。

+   `sort` 包使用反射来实现对任何类型的切片进行排序和评估的函数：`sort.Slice`、`sort.SliceStable` 和 `sort.SliceIsSorted`。

+   Go 标准库中反射的最后一个主要用途是将数据编组为 JSON 和 XML，以及在各种 `encoding` 包中定义的其他数据格式。结构标签（我将很快讨论）通过反射访问，并且结构体中的字段也使用反射进行读取和写入。

这些示例大多数有一个共同点：它们涉及访问和格式化导入到或导出出 Go 程序的数据。你经常会看到反射在你的程序和外部世界之间的边界处使用。

当你看看可以用反射实现的技术时，请记住这种力量是有代价的。使用反射比不使用它执行相同操作要慢得多。我将在 “仅在值得时使用反射” 中进一步讨论这一点。更重要的是，使用反射的代码更加脆弱和冗长。`reflect` 包中的许多函数和方法在传递错误类型的数据时会导致 panic。请确保在代码中留下注释，解释你正在做什么，以便将来的审阅者（包括你自己）能够清楚地理解。

###### 注意

Go 标准库中 `reflect` 包的另一个用途是测试。在 “Slices” 中，我提到了一个在 `reflect` 包中可以找到的函数 `DeepEqual`。它在 `reflect` 包中是因为它利用反射来完成工作。`reflect.DeepEqual` 函数检查两个值是否“深度相等”。这比使用 `==` 进行比较得到的更彻底，它被用作标准库中验证测试结果的一种方式。它还可以比较使用 `==` 无法比较的东西，如切片和映射。大多数情况下，你不需要 `DeepEqual`。自 Go 1.21 发布以来，使用 `slices.Equal` 和 `maps.Equal` 来检查切片和映射的相等性更快。

## 类型、种类和值

现在你知道什么是反射及何时需要它了，让我们看看它是如何工作的。标准库中的 `reflect` 包是实现 Go 中反射的类型和函数的家园。反射围绕三个核心概念构建：类型、种类和值。

### 类型和种类

*类型*就是它听起来的样子。它定义了变量的属性，它可以持有什么，以及你如何与它交互。通过反射，您可以使用代码查询类型以了解这些属性。

您可以使用`reflect`包中的`TypeOf`函数获取变量类型的反射表示：

```go
vType := reflect.TypeOf(v)
```

`reflect.TypeOf`函数返回一个类型为`reflect.Type`的值，表示传递给`TypeOf`函数的变量的类型。`reflect.Type`类型定义了有关变量类型信息的方法。我不能覆盖所有方法，但这里有一些。

`Name`方法返回的是类型的名称，毫不奇怪，这里有一个快速示例：

```go
var x int
xt := reflect.TypeOf(x)
fmt.Println(xt.Name())     // returns int
f := Foo{}
ft := reflect.TypeOf(f)
fmt.Println(ft.Name())     // returns Foo
xpt := reflect.TypeOf(&x)
fmt.Println(xpt.Name())    // returns an empty string
```

你从一个类型为`int`的变量`x`开始。你将它传递给`reflect.TypeOf`，然后得到一个`reflect.Type`实例。对于像`int`这样的原始类型，`Name`返回类型的名称，在这种情况下是字符串`int`对于你的`int`。对于结构体，返回结构体的名称。某些类型，如切片或指针，没有名称；在这些情况下，`Name`返回一个空字符串。

`reflect.Type`上的`Kind`方法返回一个`reflect.Kind`类型的值，它是一个常量，指示类型由切片、映射、指针、结构体、接口、字符串、数组、函数、整数或其他一些原始类型组成。种类和类型之间的差异可能会令人困惑。记住这个规则：如果您定义了一个名为`Foo`的结构体，则种类是`reflect.Struct`，类型是“Foo”。

种类非常重要。在使用反射时需要注意的一件事是，几乎`reflect`包中的所有内容都假设您知道自己在做什么。在`reflect.Type`和其他`reflect`包中定义的某些方法仅对特定种类有效。例如，在`reflect.Type`上有一个名为`NumIn`的方法。如果您的`reflect.Type`实例表示一个函数，则返回函数的输入参数数量。如果您的`reflect.Type`实例不是函数，则调用`NumIn`将导致程序恐慌。

###### 警告

一般来说，如果您调用对类型的种类无意义的方法，方法调用会引发恐慌。始终记住使用反射类型的种类来知道哪些方法可以工作，哪些方法会引发恐慌。

另一个`reflect.Type`上的重要方法是`Elem`。Go 中的一些类型具有对其他类型的引用，`Elem`是查找包含类型的方法。例如，让我们使用`reflect.TypeOf`来指向一个`int`的指针：

```go
var x int
xpt := reflect.TypeOf(&x)
fmt.Println(xpt.Name())        // returns an empty string
fmt.Println(xpt.Kind())        // returns reflect.Pointer
fmt.Println(xpt.Elem().Name()) // returns "int"
fmt.Println(xpt.Elem().Kind()) // returns reflect.Int
```

这给您一个带有空名称和`reflect.Pointer`种类的`reflect.Type`实例。当`reflect.Type`表示指针时，`Elem`返回指针指向的类型的`reflect.Type`。在这种情况下，`Name`方法返回“int”，`Kind`返回`reflect.Int`。`Elem`方法还适用于切片、映射、通道和数组。

`reflect.Type`上有一些用于反射结构体的方法。使用`NumField`方法获取结构体中字段的数量，并使用`Field`方法按索引获取结构体中的字段。该方法返回一个`reflect.StructField`，它描述了每个字段的名称、顺序、类型和结构标签。让我们看一个快速示例，您可以在[Go Playground](https://oreil.ly/Ynv_4)上运行它，或者在[第十六章仓库](https://oreil.ly/jAIdQ)的*sample_code/struct_tag*目录中运行它：

```go
type Foo struct {
    A int    `myTag:"value"`
    B string `myTag:"value2"`
}

var f Foo
ft := reflect.TypeOf(f)
for i := 0; i < ft.NumField(); i++ {
    curField := ft.Field(i)
    fmt.Println(curField.Name, curField.Type.Name(),
        curField.Tag.Get("myTag"))
}
```

您创建了一个`Foo`类型的实例，并使用`reflect.TypeOf`获取`f`的`reflect.Type`。接下来，使用`NumField`方法设置一个`for`循环以获取`f`中每个字段的索引。然后使用`Field`方法获取表示字段的`reflect.StructField`结构体，然后可以使用`reflect.StructField`上的字段获取有关字段的更多信息。此代码打印如下内容：

```go
A int value
B string value2
```

`reflect.Type`中还有许多其他方法，但它们都遵循相同的模式，允许您访问描述变量类型的信息。您可以查看标准库中的[`reflect.Type`文档](https://oreil.ly/p4AZ6)以获取更多信息。

### 值

除了检查变量的类型外，您还可以使用反射来读取变量的值、设置其值或从头开始创建新值。

您使用`reflect.ValueOf`函数创建一个表示变量值的`reflect.Value`实例：

```go
vValue := reflect.ValueOf(v)
```

由于 Go 语言中的每个变量都有一个类型，`reflect.Value`有一个叫做`Type`的方法，用于返回`reflect.Value`的`reflect.Type`。`reflect.Type`上也有一个`Kind`方法，就像在`reflect.Type`上一样。

就像`reflect.Type`有用于查找关于变量类型信息的方法一样，`reflect.Value`也有用于查找关于变量值信息的方法。我不会覆盖所有方法，但让我们看一下如何使用`reflect.Value`来获取变量的值。

我将从演示如何从`reflect.Value`中读取您的值开始。`Interface`方法将变量的值作为`any`返回。当您将`Interface`返回的值放入变量中时，您必须使用类型断言返回可用类型：

```go
s := []string{"a", "b", "c"}
sv := reflect.ValueOf(s)        // sv is of type reflect.Value
s2 := sv.Interface().([]string) // s2 is of type []string
```

虽然`Interface`可以用于包含任何类型值的`reflect.Value`实例，但如果变量的类型是内置的基本类型之一：`Bool`、`Complex`、`Int`、`Uint`、`Float`和`String`，则可以使用特殊的方法。如果变量的类型是字节切片，则`Bytes`方法也适用。如果使用不匹配`reflect.Value`类型的方法，您的代码将会恐慌。如果您不确定`Value`的类型，方法可以预先检查：`CanComplex`、`CanFloat`、`CanInt`和`CanUint`方法验证`Value`是否属于数值类型之一，`CanConvert`检查其他类型。

您也可以使用反射来设置变量的值，但这是一个三步过程。

首先，您将变量的指针传递给`reflect.ValueOf`。这将返回一个代表指针的`reflect.Value`：

```go
i := 10
iv := reflect.ValueOf(&i)
```

接下来，您需要获取要设置的实际值。您使用`reflect.Value`上的`Elem`方法来获取由传递给`reflect.ValueOf`的指针指向的值。正如`reflect.Type`上的`Elem`返回包含类型指向的类型一样，`reflect.Value`上的`Elem`返回由指针指向的值或存储在接口中的值：

```go
ivv := iv.Elem()
```

最后，您可以到达用于设置值的实际方法。正如有用于读取原始类型的特殊情况方法一样，还有用于设置原始类型的特殊情况方法：`SetBool`、`SetInt`、`SetFloat`、`SetString`和`SetUint`。在示例中，调用`ivv.SetInt(20)`会更改`i`的值。如果现在打印出`i`，您将得到 20：

```go
ivv.SetInt(20)
fmt.Println(i) // prints 20
```

对于所有其他类型，您需要使用`Set`方法，该方法接受`reflect.Value`类型的变量。您设置的值不需要是指针，因为您只是读取该值，而不是更改它。正如您可以使用`Interface()`读取原始类型一样，您可以使用`Set`写入原始类型。

将`reflect.ValueOf`传递给指针以更改输入参数的值的原因与 Go 语言中的任何其他函数类似。正如我在“指针指示可变参数”中讨论的那样，您使用指针类型的参数来指示您希望修改参数的值。当您修改该值时，您对指针进行取消引用，然后设置该值。以下两个函数遵循相同的过程：

```go
func changeInt(i *int) {
    *i = 20
}

func changeIntReflect(i *int) {
    iv := reflect.ValueOf(i)
    iv.Elem().SetInt(20)
}
```

尝试将`reflect.Value`设置为错误类型的值将导致恐慌。

###### 提示

如果未将变量的指针传递给`reflect.ValueOf`，您仍然可以使用反射读取变量的值。但是，如果尝试使用任何可以更改变量值的方法，方法调用将（不出所料地）导致恐慌。`reflect.Value`上的`CanSet`方法将告诉您调用`Set`是否会导致恐慌。

## 创建新值

在学习如何最佳使用反射之前，还有一件事需要介绍：如何创建值。`reflect.New`函数是`new`函数的反射模拟。它接受一个`reflect.Type`并返回一个指向指定类型的`reflect.Value`的指针。由于它是指针，您可以修改它，然后使用`Interface`方法将修改后的值分配给变量。

正如`reflect.New`创建标量类型的指针一样，您还可以使用反射来执行与`make`关键字相同的操作，具体方法如下：

```go
func MakeChan(typ Type, buffer int) Value

func MakeMap(typ Type) Value

func MakeMapWithSize(typ Type, n int) Value

func MakeSlice(typ Type, len, cap int) Value
```

每个函数都接受一个代表复合类型而不是包含类型的`reflect.Type`。

在构建`reflect.Type`时，必须始终从一个值开始。但是，有一个技巧可以让你创建一个变量来表示`reflect.Type`，如果没有现成的值的话：

```go
var stringType = reflect.TypeOf((*string)(nil)).Elem()

var stringSliceType = reflect.TypeOf([]string(nil))
```

变量`stringType`包含一个表示`string`的`reflect.Type`，而变量`stringSliceType`包含一个表示`[]string`的`reflect.Type`。第一行可能需要花些功夫来理解。你所做的是将`nil`转换为指向`string`的指针，使用`reflect.TypeOf`来创建该指针类型的`reflect.Type`，然后在该指针的`reflect.Type`上调用`Elem`以获取底层类型。你必须将`*string`放在括号中，因为在 Go 的运算顺序中，没有括号的话，编译器会认为你正在将`nil`转换为`string`，这是非法的。

对于`stringSliceType`来说，情况稍微简单一些，因为`nil`是切片的有效值。你只需将`nil`转换为`[]string`类型并传递给`reflect.Type`即可。

现在你有了这些类型，可以看看如何使用`reflect.New`和`reflect.MakeSlice`：

```go
ssv := reflect.MakeSlice(stringSliceType, 0, 10)

sv := reflect.New(stringType).Elem()
sv.SetString("hello")

ssv = reflect.Append(ssv, sv)
ss := ssv.Interface().([]string)
fmt.Println(ss) // prints [hello]
```

你可以在[Go Playground](https://oreil.ly/ak2PG)或[第十六章存储库](https://oreil.ly/jAIdQ)的*sample_code/reflect_string_slice*目录中自行尝试此代码。

## 使用反射检查接口值是否为 nil

正如我在“接口和 nil”中所讨论的，如果将具体类型的`nil`变量赋给接口类型的变量，则接口类型的变量不是`nil`。这是因为接口变量与类型相关联。如果要检查与接口关联的值是否为`nil`，可以使用反射的`IsValid`和`IsNil`两种方法进行检查：

```go
func hasNoValue(i any) bool {
    iv := reflect.ValueOf(i)
    if !iv.IsValid() {
        return true
    }
    switch iv.Kind() {
    case reflect.Pointer, reflect.Slice, reflect.Map, reflect.Func,
         reflect.Interface:
        return iv.IsNil()
    default:
        return false
    }
}
```

`IsValid`方法返回`true`，如果`reflect.Value`包含的是除`nil`接口之外的任何东西。你需要首先检查这一点，因为如果`IsValid`为`false`，在`reflect.Value`上调用任何其他方法都会（毫不意外地）导致恐慌。`IsNil`方法返回`true`，如果`reflect.Value`的值是`nil`，但只能在`reflect.Kind`可以是`nil`的情况下调用。如果在其零值不是`nil`的类型上调用它，会（你猜对了）导致恐慌。

你可以在[Go Playground](https://oreil.ly/D-HR9)或[第十六章存储库](https://oreil.ly/jAIdQ)的*sample_code/no_value*目录中看到此函数的使用。

即使可以检测到带有`nil`值的接口，也应努力编写代码，确保在与接口关联的值为`nil`时也能正常运行。仅在没有其他选择时使用此代码。

## 使用反射编写数据编组器

如前所述，反射是标准库用于实现编组和解组的方法。让我们看看通过构建自己的数据编组器来实现它的方法。Go 提供`csv.NewReader`和`csv.NewWriter`函数来将 CSV 文件读取到字符串的切片的切片中，并将字符串的切片的切片写入 CSV 文件，但是标准库中没有任何内容可以自动将该数据映射到结构体中的字段。以下代码将添加此缺失的功能。

###### 注意

此处的示例已经削减了一些内容以适应，减少了支持的类型数量。您可以在[The Go Playground](https://oreil.ly/VDytK)或[第十六章代码示例/csv](https://oreil.ly/jAIdQ)目录中找到完整的代码。

首先定义您的 API。与其他编组器一样，您将定义一个结构标签，指定要映射到结构体中字段的数据字段的名称：

```go
type MyData struct {
    Name   string `csv:"name"`
    Age    int    `csv:"age"`
    HasPet bool   `csv:"has_pet"`
}
```

公共 API 包括两个函数：

```go
// Unmarshal maps all of the rows of data in a slice of slice of strings
// into a slice of structs.
// The first row is assumed to be the header with the column names.
func Unmarshal(data [][]string, v any) error

// Marshal maps all of the structs in a slice of structs to a slice of slice
// of strings.
// The first row written is the header with the column names.
func Marshal(v any) ([][]string, error)
```

首先，您将从`Marshal`开始，编写该函数，然后查看它使用的两个辅助函数：

```go
func Marshal(v any) ([][]string, error) {
    sliceVal := reflect.ValueOf(v)
    if sliceVal.Kind() != reflect.Slice {
        return nil, errors.New("must be a slice of structs")
    }
    structType := sliceVal.Type().Elem()
    if structType.Kind() != reflect.Struct {
        return nil, errors.New("must be a slice of structs")
    }
    var out [][]string
    header := marshalHeader(structType)
    out = append(out, header)
    for i := 0; i < sliceVal.Len(); i++ {
        row, err := marshalOne(sliceVal.Index(i))
        if err != nil {
            return nil, err
        }
        out = append(out, row)
    }
    return out, nil
}
```

由于可以编组任何类型的结构体，因此需要使用`any`类型的参数。这不是指向结构体切片的指针，因为您只是从您的切片中读取，而不是修改它。

您的 CSV 的第一行将是包含列名的标题，因此您从结构体的字段中的结构标签获取这些列名。您使用`Type`方法从`reflect.Value`获取切片的`reflect.Type`，然后调用`Elem`方法获取切片元素的`reflect.Type`。然后将其传递给`marshalHeader`并将响应追加到您的输出。

接下来，您通过反射迭代每个结构切片中的每个元素，将每个元素的`reflect.Value`传递给`marshalOne`，并将结果追加到您的输出中。完成迭代后，返回您的`string`切片的切片。

查看您的第一个辅助函数`marshalHeader`的实现：

```go
func marshalHeader(vt reflect.Type) []string {
    var row []string
    for i := 0; i < vt.NumField(); i++ {
        field := vt.Field(i)
        if curTag, ok := field.Tag.Lookup("csv"); ok {
            row = append(row, curTag)
        }
    }
    return row
}
```

此函数简单地循环遍历`reflect.Type`的字段，读取每个字段上的`csv`标签，将其追加到`string`切片中，并返回该切片。

第二个辅助函数是`marshalOne`：

```go
func marshalOne(vv reflect.Value) ([]string, error) {
    var row []string
    vt := vv.Type()
    for i := 0; i < vv.NumField(); i++ {
        fieldVal := vv.Field(i)
        if _, ok := vt.Field(i).Tag.Lookup("csv"); !ok {
            continue
        }
        switch fieldVal.Kind() {
        case reflect.Int:
            row = append(row, strconv.FormatInt(fieldVal.Int(), 10))
        case reflect.String:
            row = append(row, fieldVal.String())
        case reflect.Bool:
            row = append(row, strconv.FormatBool(fieldVal.Bool()))
        default:
            return nil, fmt.Errorf("cannot handle field of kind %v",
                                   fieldVal.Kind())
        }
    }
    return row, nil
}
```

此函数接受一个`reflect.Value`并返回一个`string`切片。您创建`string`切片，并针对结构体中的每个字段，使用其`reflect.Kind`进行切换以确定如何将其转换为`string`，并将其追加到输出中。

您的简单编组器现在已经完成。让我们看看您需要做什么来解组：

```go
func Unmarshal(data [][]string, v any) error {
    sliceValPointer := reflect.ValueOf(v)
    if sliceValPointer.Kind() != reflect.Pointer {
        return errors.New("must be a pointer to a slice of structs")
    }
    sliceVal := sliceValPointer.Elem()
    if sliceVal.Kind() != reflect.Slice {
        return errors.New("must be a pointer to a slice of structs")
    }
    structType := sliceVal.Type().Elem()
    if structType.Kind() != reflect.Struct {
        return errors.New("must be a pointer to a slice of structs")
    }

    // assume the first row is a header
    header := data[0]
    namePos := make(map[string]int, len(header))
    for i, name := range header {
        namePos[name] = i
    }

    for _, row := range data[1:] {
        newVal := reflect.New(structType).Elem()
        err := unmarshalOne(row, namePos, newVal)
        if err != nil {
            return err
        }
        sliceVal.Set(reflect.Append(sliceVal, newVal))
    }
    return nil
}
```

由于您正在将数据复制到任意类型的结构体切片中，因此需要使用`any`类型的参数。此外，由于您正在修改存储在此参数中的值，*必须*传递指向结构体切片的指针。`Unmarshal`函数将该结构体切片指针转换为`reflect.Value`，然后获取底层切片，并获取底层切片中结构体的类型。

如我先前所说，该代码假定数据的第一行是包含列名的标题。您可以利用这些信息构建映射，从而将`csv`结构标签的值与正确的数据元素关联起来。

然后，您循环遍历所有剩余的`string`切片，使用结构体的`reflect.Type`创建一个新的`reflect.Value`，调用`unmarshalOne`将当前`string`切片中的数据复制到结构体中，然后将结构体添加到您的切片中。在遍历完所有数据行后，返回。

唯一剩下的是查看`unmarshalOne`的实现：

```go
func unmarshalOne(row []string, namePos map[string]int, vv reflect.Value) error {
    vt := vv.Type()
    for i := 0; i < vv.NumField(); i++ {
        typeField := vt.Field(i)
        pos, ok := namePos[typeField.Tag.Get("csv")]
        if !ok {
            continue
        }
        val := row[pos]
        field := vv.Field(i)
        switch field.Kind() {
        case reflect.Int:
            i, err := strconv.ParseInt(val, 10, 64)
            if err != nil {
                return err
            }
            field.SetInt(i)
        case reflect.String:
            field.SetString(val)
        case reflect.Bool:
            b, err := strconv.ParseBool(val)
            if err != nil {
                return err
            }
            field.SetBool(b)
        default:
            return fmt.Errorf("cannot handle field of kind %v",
                              field.Kind())
        }
    }
    return nil
}
```

此函数迭代新创建的`reflect.Value`中的每个字段，使用当前字段上的`csv`结构标签查找其名称，在`data`切片中使用`namePos`映射查找元素，将值从`string`转换为正确的类型，并设置当前字段的值。在填充完所有字段后，函数返回。

现在您已经编写了自己的编组器和解组器，可以与 Go 标准库中现有的 CSV 支持集成：

```go
data := `name,age,has_pet
Jon,"100",true
"Fred ""The Hammer"" Smith",42,false
Martha,37,"true"
`
r := csv.NewReader(strings.NewReader(data))
allData, err := r.ReadAll()
if err != nil {
    panic(err)
}
var entries []MyData
Unmarshal(allData, &entries)
fmt.Println(entries)

//now to turn entries into output
out, err := Marshal(entries)
if err != nil {
    panic(err)
}
sb := &strings.Builder{}
w := csv.NewWriter(sb)
w.WriteAll(out)
fmt.Println(sb)
```

## 使用反射构建函数以自动化重复任务

Go 另一个使用反射的功能是创建函数。您可以使用此技术来为任何传入的函数添加常见功能，而无需编写重复的代码。例如，这是一个工厂函数的示例，它为任何传入的函数添加计时功能：

```go
func MakeTimedFunction(f any) any {
    ft := reflect.TypeOf(f)
    fv := reflect.ValueOf(f)
    wrapperF := reflect.MakeFunc(ft, func(in []reflect.Value) []reflect.Value {
        start := time.Now()
        out := fv.Call(in)
        end := time.Now()
        fmt.Println(end.Sub(start))
        return out
    })
    return wrapperF.Interface()
}
```

此函数接受任何函数作为参数，因此参数的类型是`any`。然后，它将代表该函数的`reflect.Type`传递给`reflect.MakeFunc`，还传递一个捕获了起始时间的闭包，使用反射调用原始函数，捕获结束时间，打印出差异，并返回原始函数计算的值。从`reflect.MakeFunc`返回的值是一个`reflect.Value`，您调用其`Interface`方法以获取要返回的值。以下是如何使用它的示例：

```go
func timeMe(a int) int {
    time.Sleep(time.Duration(a) * time.Second)
    result := a * 2
    return result
}

func main() {
    timed:= MakeTimedFunction(timeMe).(func(int) int)
    fmt.Println(timed(2))
}
```

您可以在[Go Playground](https://oreil.ly/NDfp1)上或在[第十六章存储库](https://oreil.ly/jAIdQ)的*sample_code/timed_function*目录中运行此程序的更完整版本。

尽管生成函数很聪明，但在使用此功能时要小心。确保清楚您何时使用生成的函数以及它添加了什么功能。否则，您将使程序中数据流的理解变得更加困难。此外，正如我将在“仅在值得时使用反射”中讨论的那样，反射会使您的程序变慢，因此，除非您正在生成的代码已经执行缓慢操作（如网络调用），否则使用它来生成和调用函数会严重影响性能。请记住，反射在最佳情况下用于将数据映射到程序的边缘。

遵循这些规则生成函数的一个项目是我的 SQL 映射库 Proteus。它通过从 SQL 查询和函数字段或变量生成函数来创建类型安全的数据库 API。你可以在我的 GopherCon 2017 演讲中了解更多关于 Proteus 的信息，[“Runtime Generated, Typesafe, and Declarative: Pick Any Three,”](https://oreil.ly/ZUE47)，并且你可以在 [GitHub](https://oreil.ly/KtFyj) 上找到源代码。

## 你可以使用反射构建结构体，但最好不要这样做

你可以用反射做一件更怪异的事情。`reflect.StructOf` 函数接受 `reflect.StructField` 切片并返回代表新结构体类型的 `reflect.Type`。这些结构体只能分配给 `any` 类型的变量，并且只能使用反射读取和写入它们的字段。

大多数情况下，这只是学术兴趣的一个特性。要查看 `reflect.StructOf` 如何工作的演示，请查看 [The Go Playground](https://oreil.ly/iJwqv) 上的记忆函数或 [Chapter 16 repository](https://oreil.ly/jAIdQ) 的 *sample_code/memoizer* 目录。它使用动态生成的结构体作为映射的键来缓存函数的输出。

## 反射无法创建方法

你已经看到反射可以做的所有事情，但有一件事情它做不到。虽然你可以使用反射来创建新函数和新的结构体类型，但没有办法使用反射向类型添加方法。这意味着你不能使用反射来创建一个实现接口的新类型。

## 只有在值得时才使用反射

虽然反射在转换 Go 边界处的数据时至关重要，但在其他情况下使用时要小心。反射并非免费。为了演示，让我们使用反射来实现 `Filter`。你在 “Generic Functions Abstract Algorithms” 中看过一个通用实现，但现在让我们看看基于反射的版本是什么样子：

```go
func Filter(slice any, filter any) any {
    sv := reflect.ValueOf(slice)
    fv := reflect.ValueOf(filter)

    sliceLen := sv.Len()
    out := reflect.MakeSlice(sv.Type(), 0, sliceLen)
    for i := 0; i < sliceLen; i++ {
        curVal := sv.Index(i)
        values := fv.Call([]reflect.Value{curVal})
        if values[0].Bool() {
            out = reflect.Append(out, curVal)
        }
    }
    return out.Interface()
}
```

你可以像这样使用它：

```go
names := []string{"Andrew", "Bob", "Clara", "Hortense"}
longNames := Filter(names, func(s string) bool {
    return len(s) > 3
}).([]string)
fmt.Println(longNames)

ages := []int{20, 50, 13}
adults := Filter(ages, func(age int) bool {
    return age >= 18
}).([]int)
fmt.Println(adults)
```

这将打印出以下内容：

```go
[Andrew Clara Hortense]
[20 50]
```

使用反射的过滤函数并不难理解，但它肯定比自定义编写的或通用函数更长。让我们看看在 Go 1.20 上，使用 16 GB RAM 的 Apple Silicon M1，当过滤包含 1,000 个字符串和`int`元素的切片时，与自定义编写的函数相比它的表现如何：

```go
BenchmarkFilterReflectString-8     5870  203962 ns/op  46616 B/op  2219 allocs/op
BenchmarkFilterGenericString-8   294355    3920 ns/op  16384 B/op     1 allocs/op
BenchmarkFilterString-8          302636    3885 ns/op  16384 B/op     1 allocs/op
BenchmarkFilterReflectInt-8        5756  204530 ns/op  45240 B/op  2503 allocs/op
BenchmarkFilterGenericInt-8      439100    2698 ns/op   8192 B/op     1 allocs/op
BenchmarkFilterInt-8             443745    2677 ns/op   8192 B/op     1 allocs/op

```

你可以在 [Chapter 16 repository](https://oreil.ly/jAIdQ) 的 *sample_code/reflection_filter* 目录中找到此代码，因此你可以自行运行它。

此基准测试展示了在可以使用泛型时使用泛型的价值。反射的性能比自定义或通用函数慢 50 多倍，对于 `int` 来说大约慢 75 倍。它使用的内存显著更多，并执行数千次分配，这为垃圾收集器创建了额外工作。泛型版本提供了与自定义编写的函数相同的性能，而无需编写多个版本。

反射实现的一个更严重的缺点是，编译器无法阻止您为“切片”或“过滤器”参数传递错误类型。您可能不介意花费几千个纳秒的 CPU 时间，但如果有人向“过滤器”传递错误类型的函数或切片，则您的程序将在生产中崩溃。维护成本可能太高以致接受。

有些东西无法使用泛型编写，你需要回退到反射（reflection）机制。CSV 的编组和解组需要反射，内存化程序也需要。在这两种情况下，你需要处理不同（和未知）类型的未知数量的值。但在使用反射之前，请确保它是必不可少的。

# `unsafe`即不安全

就像`reflect`包允许您操纵类型和值一样，`unsafe`包允许您操纵内存。`unsafe`包非常小，也非常奇怪。它定义了几个函数和一个类型，都不像其他包中的类型和函数。

鉴于 Go 对内存安全的重视，你可能会想为什么`unsafe`还存在。就像你使用`reflect`在外部世界和 Go 代码之间转换文本数据一样，你使用`unsafe`来转换二进制数据。使用`unsafe`的两个主要原因。Diego Elias Costa 等人 2020 年的一篇论文名为[“Breaking Type-Safety in Go: An Empirical Study on the Usage of the unsafe Package”](https://oreil.ly/N_6JX)调查了 2,438 个流行的 Go 开源项目，并发现了以下问题：

+   在研究的 Go 项目中，有 24%的项目至少在其代码库中使用了`unsafe`。

+   大多数`unsafe`用途是为了与操作系统和 C 代码集成。

+   开发者经常使用`unsafe`来编写更高效的 Go 代码。

`unsafe`的多数用途是用于系统互操作性。Go 标准库使用`unsafe`从操作系统读取数据和向其写入数据。您可以在标准库的`syscall`包或更高级别的[`sys`包](https://oreil.ly/ueHY3)中看到示例。您可以通过 Matt Layher 撰写的[一篇很棒的博文](https://oreil.ly/VtE1t)了解如何使用`unsafe`与操作系统通信的更多信息。

`unsafe.Pointer`类型是一个特殊类型，只有一个目的：可以将任何类型的指针转换为或从`unsafe.Pointer`。除了指针之外，`unsafe.Pointer`还可以转换为或从特殊整数类型`uintptr`。与任何其他整数类型一样，您可以对其进行数学运算。这允许您进入类型的实例，提取单个字节。您还可以执行指针算术运算，就像在 C 和 C++中可以对指针进行的一样。这种字节操作会改变变量的值。

在`unsafe`代码中存在两种常见模式。第一种是在通常不可转换的两种变量类型之间进行转换。这通过在中间使用`unsafe.Pointer`进行一系列类型转换来完成。第二种是通过将变量转换为`unsafe.Pointer`，然后将`unsafe.Pointer`转换为指针，以读取或修改变量中的字节，并复制或操作其基础字节。这两种技术都要求你了解正在操作的数据的大小（以及可能的位置）。`unsafe`包中的`Sizeof`和`Offsetof`函数提供了这些信息。

## 使用 `Sizeof` 和 `Offsetof`

`unsafe`中的一些函数揭示了各种类型在内存中的布局方式。你将首先看到的是`Sizeof`。顾名思义，它返回传递给它的任何内容的字节大小。

尽管数值类型的大小相对明显（比如`int16`是 16 位或 2 字节，`byte`是 1 字节等），其他类型就复杂一些了。对于指针而言，你得到的是用于存储指针的内存大小（在 64 位系统上通常为 8 字节），而不是指针指向的数据的大小。这就是为什么在 64 位系统上，`Sizeof` 认为任何切片的长度为 24 字节；它实际上由两个`int`字段（用于长度和容量）和指向切片数据的指针组成。在 64 位系统上，任何字符串的长度为 16 字节（包括一个长度`int`和指向字符串内容的指针）。在 64 位系统上，任何`map`都是 8 字节，因为在 Go 运行时中，`map`被实现为指向一个相当复杂的数据结构的指针。

数组是值类型，因此其大小通过将数组长度乘以数组中每个元素的大小来计算。

对于结构体而言，其大小是字段大小的总和，加上*对齐*的一些调整。计算机喜欢以固定大小的块读写数据，而且它们确实不希望一个值从一个块开始到另一个块结束。为了实现这一点，编译器在字段之间添加填充，以便它们正确对齐。编译器还希望整个结构体能够正确对齐。在 64 位系统上，它将在结构体末尾添加填充，使其大小达到 8 字节的倍数。

`unsafe`中的另一个函数`Offsetof`告诉您结构体中字段的位置。让我们使用`Sizeof`和`Offsetof`来看看字段顺序对结构体大小的影响。假设你有两个结构体：

```go
type BoolInt struct {
    b bool
    i int64
}

type IntBool struct {
    i int64
    b bool
}
```

运行这段代码

```go
fmt.Println(unsafe.Sizeof(BoolInt{}),
    unsafe.Offsetof(BoolInt{}.b),
    unsafe.Offsetof(BoolInt{}.i))

fmt.Println(unsafe.Sizeof(IntBool{}),
    unsafe.Offsetof(IntBool{}.i),
    unsafe.Offsetof(IntBool{}.b))
```

产生以下输出：

```go
16 0 8
16 0 8
```

两种类型的大小都是 16 字节。当`bool`放在前面时，编译器在`b`和`i`之间放置 7 字节的填充。当`int64`放在前面时，编译器在`b`后面放置 7 字节的填充，以使结构体的大小成为 8 字节的倍数。

当存在更多字段时，你可以看到字段顺序的影响：

```go
type BoolIntBool struct {
    b  bool
    i  int64
    b2 bool
}

type BoolBoolInt struct {
    b  bool
    b2 bool
    i  int64
}

type IntBoolBool struct {
    i  int64
    b  bool
    b2 bool
}
```

打印出这些类型的大小和偏移量将产生以下输出：

```go
24 0 8 16
16 0 1 8
16 0 8 9
```

将`int64`字段放在两个`bool`字段之间会产生一个 24 字节长的结构体，因为两个`bool`字段需要填充为 8 字节。而将`bool`字段分组在一起会产生一个 16 字节长的结构体，与仅有两个字段的结构体相同。你可以在[The Go Playground](https://oreil.ly/1X9--)或[第十六章代码库](https://oreil.ly/jAIdQ)的*sample_code/sizeof_offsetof*目录中验证这一点。

虽然大部分时间仅仅是学术兴趣，但这些信息在两种情况下都很有用。第一种情况是在管理大量数据的程序中。有时，通过重新排列结构体中经常使用的字段的顺序，以尽量减少对齐所需的填充量，可以显著节省内存使用。

第二种情况发生在你想要直接将一系列字节映射到一个结构体时。接下来你会看到这个情况。

## 使用 unsafe 转换外部二进制数据

正如前面提到的，人们使用`unsafe`的主要原因之一是为了性能，特别是在从网络读取数据时。如果你想将数据映射到或从 Go 数据结构中映射出来，`unsafe.Pointer`提供了一种非常快速的方式。你可以通过一个构造的例子来探索这一点。想象一个*wire protocol*（一个规范，指示在网络通信时以哪种顺序写入哪些字节）具有以下结构：

+   Value: 4 字节，表示无符号的大端 32 位整数

+   Label: 10 字节，值的 ASCII 名称

+   Active: 1 字节，布尔标志，指示字段是否激活

+   Padding: 1 字节，因为你希望所有内容都适应 16 字节

###### 注意

通过网络发送的数据通常以大端格式发送（最重要的字节先），通常称为*network byte order*。由于今天使用的大多数 CPU 都是小端（或以小端模式运行的双端），因此在读取或写入数据到网络时需要小心。

这个例子的代码位于[第十六章代码库](https://oreil.ly/jAIdQ)中的*sample_code/unsafe_data*目录中。

你定义一个数据结构，其内存布局与 wire protocol 匹配：

```go
type Data struct {
    Value  uint32   // 4 bytes
    Label  [10]byte // 10 bytes
    Active bool     // 1 byte
    // Go pads this with 1 byte to make it align
}
```

你可以使用`unsafe.Sizeof`定义一个代表其大小的常量：

```go
const dataSize = unsafe.Sizeof(Data{}) // sets dataSize to 16
```

（关于`unsafe.Sizeof`和`Offsetof`的奇怪之处之一是，你可以在`const`表达式中使用它们。数据结构在内存中的大小和布局在编译时已知，因此这些函数的结果在编译时计算，就像一个常量表达式一样。）

假设你刚刚从网络上读取了以下字节：

```go
[0 132 95 237 80 104 111 110 101 0 0 0 0 0 1 0]
```

你将把这些字节读入长度为 16 的数组中，然后将该数组转换为前面描述的`struct`。

使用安全的 Go 代码，你可以这样映射它：

```go
func DataFromBytes(b [dataSize]byte) Data {
    d := Data{}
    d.Value = binary.BigEndian.Uint32(b[:4])
    copy(d.Label[:], b[4:14])
    d.Active = b[14] != 0
    return d
}
```

或者，你可以使用`unsafe.Pointer`代替：

```go
func DataFromBytesUnsafe(b [dataSize]byte) Data {
    data := *(*Data)(unsafe.Pointer(&b))
    if isLE {
        data.Value = bits.ReverseBytes32(data.Value)
    }
    return data
}
```

第一行有点令人困惑，但你可以分开来理解发生了什么。首先，你获取字节数组的地址并将其转换为`unsafe.Pointer`。然后将`unsafe.Pointer`转换为`(*Data)`（由于 Go 的操作顺序，你必须将`(*Data)`放在括号中）。你想要返回结构体而不是指向它的指针，因此你对指针进行解引用。接下来，你检查标志来看你是否处于小端平台。如果是，你会反转`Value`字段中的字节。最后，你返回该值。

###### 注

为什么你在参数上使用数组而不是切片？记住，数组和结构体一样，都是值类型；字节直接分配。这意味着你可以直接将`b`中的值映射到`data`结构体中。切片由三部分组成：长度、容量和指向实际值的指针。稍后你会看到如何使用`unsafe`将切片映射到结构体中。

你怎么知道你在小端平台上？这是你正在使用的代码：

```go
var isLE bool

func init() {
    var x uint16 = 0xFF00
    xb := *(*[2]byte)(unsafe.Pointer(&x))
    isLE = (xb[0] == 0x00)
}
```

正如我在“尽可能避免使用 init 函数”中所讨论的，除了初始化包级别的值（其值实际上是不可变的）之外，你应该避免使用`init`函数。由于处理器的字节序在程序运行时不会改变，这是一个很好的使用情况。

在小端平台上，表示`x`的字节将存储为[00 FF]。在大端平台上，`x`存储在内存中为[FF 00]。你使用`unsafe.Pointer`将数字转换为字节数组，并检查第一个字节来确定`isLE`的值。

同样，如果你想将你的`Data`写回网络，你可以使用安全的 Go：

```go
func BytesFromData(d Data) [dataSize]byte {
    out := [dataSize]byte{}
    binary.BigEndian.PutUint32(out[:4], d.Value)
    copy(out[4:14], d.Label[:])
    if d.Active {
        out[14] = 1
    }
    return out
}
```

或者你可以使用`unsafe`：

```go
func BytesFromDataUnsafe(d Data) [dataSize]byte {
    if isLE {
        d.Value = bits.ReverseBytes32(d.Value)
    }
    b := *(*[dataSize]byte)(unsafe.Pointer(&d))
    return b
}
```

如果字节存储在切片中，你可以使用`unsafe.Slice`函数从`Data`的内容创建一个切片。`unsafe.SliceData`函数用于从切片中存储的数据创建`Data`实例：

```go
func BytesFromDataUnsafeSlice(d Data) []byte {
    if isLE {
        d.Value = bits.ReverseBytes32(d.Value)
    }
    bs := unsafe.Slice((*byte)(unsafe.Pointer(&d)), dataSize)
    return bs
}

func DataFromBytesUnsafeSlice(b []byte) Data {
    data := *(*Data)((unsafe.Pointer)(unsafe.SliceData(b)))
    if isLE {
        data.Value = bits.ReverseBytes32(data.Value)
    }
    return data
}
```

`unsafe.Slice`的第一个参数需要两次转换。第一次转换将`Data`实例的指针转换为`unsafe.Pointer`。然后你需要再次转换为你想要切片保存的数据类型的指针。对于字节切片，你使用`*byte`。第二个参数是数据的长度。

`unsafe.SliceData`函数接受一个切片并返回指向切片中数据类型的指针。在本例中，你传入了`[]byte`，它返回一个`*byte`。然后你使用`unsafe.Pointer`作为`*byte`和`*Data`之间的桥梁，将切片的内容转换为`Data`实例。

在 Apple Silicon M1（小端）上的计时情况如何？

```go
BenchmarkBytesFromData-8             548607271  2.185 ns/op   0 B/op 0 allocs/op
BenchmarkBytesFromDataUnsafe-8      1000000000  0.8418 ns/op  0 B/op 0 allocs/op
BenchmarkBytesFromDataUnsafeSlice-8  91179056  13.14 ns/op   16 B/op 1 allocs/op
BenchmarkDataFromBytes-8             538443861  2.186 ns/op   0 B/op 0 allocs/op
BenchmarkDataFromBytesUnsafe-8      1000000000  1.160 ns/op   0 B/op 0 allocs/op
BenchmarkDataFromBytesUnsafeSlice-8 1000000000  0.9617 ns/op  0 B/op 0 allocs/op
```

这张图中有两件事情很突出。首先，从结构体到切片的转换明显是最慢的操作，而且它是唯一分配内存的操作。这并不奇怪，因为切片的数据需要在返回函数时逃逸到堆上。在堆上分配内存几乎总是比在栈上使用内存慢。不过，从切片到结构体的转换非常快。

如果你正在处理数组，使用`unsafe`比标准方法快约 2 到 2.5 倍。如果你的程序有许多这种类型的转换，或者你尝试映射一个非常大且复杂的数据结构，使用这些低级技术是值得的。但对于绝大多数程序来说，请坚持使用安全的代码。

## 访问未导出的字段

这里有另一种你可以使用`unsafe`的魔法，但这只能作为最后的手段使用。你可以结合反射和`unsafe`来读取和修改结构体中未导出的字段。让我们看看怎么做。

首先，在一个包中定义一个结构体：

```go
type HasUnexportedField struct {
    A int
    b bool
    C string
}
```

通常，该包外的代码无法访问`b`。然而，让我们看看通过使用`unsafe`你能从另一个包中做些什么：

```go
func SetBUnsafe(huf *one_package.HasUnexportedField) {
    sf, _ := reflect.TypeOf(huf).Elem().FieldByName("b")
    offset := sf.Offset
    start := unsafe.Pointer(huf)
    pos := unsafe.Add(start, offset)
    b := (*bool)(pos)
    fmt.Println(*b) // read the value
    *b = true       // write the value
}
```

你可以使用反射访问字段`b`的类型信息。`FieldByName`方法为结构体上的任何字段返回一个`reflect.StructField`实例，即使是未导出的字段也可以。该实例包括其关联字段的偏移量。然后，你将`huf`转换为`unsafe.Pointer`，并使用`unsafe.Add`方法将偏移量添加到指针中，以找到结构体中`b`的位置。最后，将`Add`返回的`unsafe.Pointer`强制转换为`*bool`。现在，你可以读取`b`的值或设置其值。你可以在[第十六章的代码库](https://oreil.ly/jAIdQ)的*sample_code/unexported_field_access*目录中尝试此代码。

## 使用不安全工具

Go 是一个重视工具的语言，编译器标志可以帮助你找到`Pointer`和`unsafe.Pointer`的误用。使用标志`-gcflags=-d=checkptr`在运行时添加额外的检查。与竞态检查器一样，它不能保证找到每一个`unsafe`问题，并且会减慢你的程序。然而，在测试代码时这是一个良好的实践。

如果你想了解更多关于`unsafe`的内容，请阅读[文档](https://oreil.ly/xmihF)。

###### 警告

`unsafe`包非常强大且底层！除非你知道你在做什么并且需要它提供的性能改进，否则请避免使用它。

# Cgo 是用于集成，而非性能优化

就像反射和`unsafe`一样，`cgo`在 Go 程序与外界之间的边界上最为有用。反射帮助集成外部文本数据，`unsafe`最适用于操作系统和网络数据，而`cgo`则适合与 C 库集成。

尽管已经接近 50 年的历史，C 仍然是编程语言的*通用语言*。所有主要操作系统主要都是用 C 或 C++ 编写的，这意味着它们捆绑了用 C 编写的库。这也意味着几乎每种编程语言都提供了一种与 C 库集成的方式。Go 将其对 C 的外部函数接口（FFI）称为 `cgo`。

正如你已经多次看到的那样，Go 是一种偏爱显式规范的语言。Go 开发人员有时会嘲笑其他语言中的自动行为为“魔法”。然而，使用`cgo`有点像与梅林共度时光。让我们来看看这段神奇的粘合代码。

你将从一个非常简单的程序开始，调用 C 代码来进行一些数学运算。源代码在 GitHub 的 *sample_code/call_c_from_go* 目录中，在[第十六章存储库](https://oreil.ly/jAIdQ)中。首先，这是 *main.go* 中的代码：

```go
package main

import "fmt"

/*
 #cgo LDFLAGS: -lm
 #include <stdio.h>
 #include <math.h>
 #include "mylib.h"

 int add(int a, int b) {
 int sum = a + b;
 printf("a: %d, b: %d, sum %d\n", a, b, sum);
 return sum;
 }
*/
import "C"

func main() {
    sum := C.add(3, 2)
    fmt.Println(sum)
    fmt.Println(C.sqrt(100))
    fmt.Println(C.multiply(10, 20))
}
```

*mylib.h* 头文件与你的 *main.go* 在同一个目录中：

```go
int multiply(int a, int b);
```

还有一个 *mylib.c*：

```go
#include "mylib.h"

int multiply(int a, int b) {
    return a * b;
}
```

假设你的计算机上已安装了 C 编译器，你只需使用`go build`编译你的程序：

```go
$ go build
$ ./call_c_from_go
a: 3, b: 2, sum 5
5
10
200
```

这里发生了什么？标准库中没有一个真正的名为`C`的包。相反，`C` 是一个自动生成的包，其标识符大多来自于紧随其后的注释中嵌入的 C 代码。在这个例子中，你声明了一个名为`add`的 C 函数，`cgo`将其作为`C.add`的名称提供给你的 Go 程序。你还可以调用通过头文件从库中导入到注释块中的函数或全局变量，就像你从`main`中调用`C.sqrt`（从 *math.h* 导入）或`C.multiply`（从 *mylib.h* 导入）时所看到的那样。

除了出现在注释块中的标识符名称（或被导入到注释块中的名称）外，`C` 伪包还定义了像`C.int`和`C.char`这样的类型，用于表示内置的 C 类型和函数，比如`C.CString`用于将 Go 字符串转换为 C 字符串。

你可以使用更多的魔法来从 C 函数调用 Go 函数。通过在函数前放置一个`//export`注释，可以将 Go 函数暴露给 C 代码。你可以在[第十六章存储库](https://oreil.ly/jAIdQ)的 *sample_code/call_go_from_c* 目录中看到这种用法。在 *main.go* 中，你导出了 doubler 函数：

```go
//export doubler
func doubler(i int) int {
    return i * 2
}
```

当你导出一个 Go 函数时，在`import "C"`语句之前的注释中就不能再直接编写 C 代码了。你只能列出函数头部：

```go
/*
 extern int add(int a, int b);
*/
import "C"
```

将你的 C 代码放入与你的 Go 代码相同目录的一个 *.c* 文件中，并包含神奇的头文件`"cgo_export.h"`。你可以在 *example.c* 文件中看到这一点：

```go
#include "_cgo_export.h"

int add(int a, int b) {
    int doubleA = doubler(a);
    int sum = doubleA + b;
    return sum;
}
```

使用`go build`运行这个程序会得到以下输出：

```go
$ go build
$ ./call_go_from_c
8
```

到目前为止，这似乎很简单，但在使用`cgo`时会遇到一个主要的障碍：Go 是一种垃圾收集语言，而 C 不是。这使得将复杂的 Go 代码与 C 集成变得困难。虽然你可以将指针传递给 C 代码，但不能直接传递包含指针的东西。这是非常限制性的，因为像字符串、切片和函数这样的东西都是用指针实现的，因此不能包含在传递给 C 函数的结构体中。

这还不是全部：C 函数不能存储在函数返回后持续存在的 Go 指针的副本。如果你违反了这些规则，你的程序会编译和运行，但在指针指向的内存被垃圾收集时可能会崩溃或表现不正确。

如果你发现自己需要从 Go 向 C 传递包含指针的类型实例，然后再次返回到 Go，你可以使用`cgo.Handle`来包装该实例。这里有一个简短的示例。你可以在[第十六章的代码库](https://oreil.ly/jAIdQ)的*sample_code/handle*目录中找到源代码。

首先是你的 Go 代码：

```go
package main

/*
 #include <stdint.h>

 extern void in_c(uintptr_t handle);
*/
import "C"

import (
    "fmt"
    "runtime/cgo"
)

type Person struct {
    Name string
    Age  int
}

func main() {
    p := Person{
        Name: "Jon",
        Age:  21,
    }
    C.in_c(C.uintptr_t(cgo.NewHandle(p)))
}

//export processor
func processor(handle C.uintptr_t) {
    h := cgo.Handle(handle)
    p := h.Value().(Person)
    fmt.Println(p.Name, p.Age)
    h.Delete()
}
```

这是 C 代码：

```go
#include <stdint.h>
#include "_cgo_export.h"

void in_c(uintptr_t handle) {
    processor(handle);
}
```

在这段 Go 代码中，你将一个`Person`实例传递给 C 函数`in_c`。然后这个函数调用 Go 函数`processor`。通过`cgo`安全地将`Person`传递给 C 是不可能的，因为它的字段之一是一个`string`，而每个`string`都包含一个指针。为了使这个工作正常，使用`cgo.NewHandle`函数将`p`转换为`cgo.Handle`。然后将`Handle`转换为`C.uintptr_t`类型，以便将其传递给 C 函数`in_c`。该函数接受一个类型为`uintptr_t`的单一参数，这是一个类似于 Go 的`uintptr`类型的 C 类型。`in_c`函数调用 Go 函数`process`，也接受一个类型为`C.uintptr_t`的单一参数。它将此参数转换为`cgo.Handle`，然后使用类型断言将`Handle`转换为`Person`实例。你打印出`p`中的字段。现在你使用完`Handle`后，调用`Delete`方法来删除它。

除了处理指针之外，还存在其他`cgo`的限制。例如，你不能使用`cgo`调用可变参数的 C 函数（比如`printf`）。C 中的联合类型转换为字节数组。你不能调用 C 函数指针（但可以将其分配给一个 Go 变量并将其传递回 C 函数）。

这些规则使得使用`cgo`并不简单。如果你有 Python 或 Ruby 的背景，你可能认为出于性能原因使用`cgo`是值得的。这些开发者将其程序的性能关键部分写在 C 中。NumPy 的速度归功于 Python 代码封装的 C 库。

在大多数情况下，Go 代码比 Python 或 Ruby 快几倍，因此大大减少了需要用低级语言重写算法的必要性。你可能会认为在需要额外性能提升时可以保存`cgo`，但不幸的是，使用`cgo`使代码加速并不容易。由于处理和内存模型的不匹配，从 Go 调用 C 函数比从 C 函数调用另一个 C 函数慢大约 29 倍。

在 CapitalGo 2018 年，Filippo Valsorda 发表了名为“为什么 cgo 很慢”的演讲。不幸的是，演讲没有录制，但[幻灯片可用](https://oreil.ly/MLRFY)。它们解释了为什么`cgo`很慢以及为什么未来不会显著加快其速度。在他的博客文章[“Go 1.21 中 CGO 性能”](https://oreil.ly/AoCDG)中，Shane Hansen 测量了在 Intel Core i7-12700H 上使用 Go 1.21 进行`cgo`调用的开销大约为 40 纳秒。

###### 提示

由于`cgo`速度不快，且对于复杂程序不易使用，使用`cgo`的唯一理由是必须使用 C 库且没有适当的 Go 替代品。与其自己编写`cgo`，不如看看是否有第三方模块提供了包装器。例如，如果要在 Go 应用程序中嵌入 SQLite，请查看[GitHub](https://oreil.ly/IEskN)。对于 ImageMagick，请查看[此存储库](https://oreil.ly/l58-1)。

如果你发现自己需要使用一个没有包装器的内部 C 库或第三方库，可以在 Go 文档中找到[更多详情](https://oreil.ly/9JvNI)。有关在使用`cgo`时可能遇到的性能和设计折衷的信息，请阅读 Tobias Grieger 的博客文章[“Cgo 的成本和复杂性”](https://oreil.ly/Oj9Tw)。

# 练习

现在是时候测试你自己是否可以使用反射、`unsafe`和`cgo`编写小程序了。练习的答案可以在[第十六章存储库](https://oreil.ly/jAIdQ)的*exercise_solutions*目录中找到。

1.  使用反射创建一个简单的最小字符串长度验证器来验证结构字段。编写一个`ValidateStringLength`函数，接受一个结构体并在字段是字符串、具有名为`minStrlen`的结构标签且字段值长度小于结构标签中指定的值时返回错误。忽略非字符串字段和没有`minStrlen`结构标签的字符串字段。使用`errors.Join`报告所有无效字段。确保验证传递的是结构体。如果所有字段长度正确，则返回`nil`。

1.  使用`unsafe.Sizeof`和`unsafe.Offsetof`打印出[*ch16/tree/main/sample_code/orders*](https://oreil.ly/SrSpa)中定义的`OrderInfo`结构体的大小和偏移量。创建一个新类型`SmallOrderInfo`，其字段相同，但重新排序以尽可能减少内存使用。

1.  复制 [*ch16/tree/main/sample_code/mini_calc*](https://oreil.ly/E-PfQ) 中的 C 代码到你自己的模块，并使用`cgo`从 Go 程序中调用它。

# 结语

在本章中，你学习了反射（reflection）、`unsafe`和`cgo`。这些特性可能是 Go 语言最令人兴奋的部分，因为它们允许你打破 Go 语言无趣、类型安全、内存安全的规则。更重要的是，你学会了*为什么*要打破这些规则，以及为什么大多数时候应该避免这样做。

你已经完成了对 Go 语言及其惯用法的学习之旅。就像任何毕业典礼一样，现在是总结的时候了。让我们回顾一下序言中的内容。"[P]roperly written, Go is boring…​.Well-written Go programs tend to be straightforward and sometimes a bit repetitive." 希望你现在能明白为什么这会导致更好的软件工程。

惯用的 Go 语言是一组工具、实践和模式，使得随着时间和团队变化更容易维护软件。这并不是说其他语言的文化不重视可维护性；只是可能不是它们的首要考虑因素。它们更强调性能、新特性或简洁的语法。这些权衡是有存在的必要性，但从长远来看，我相信 Go 语言专注于打造持久软件的焦点最终会胜出。祝你在为未来 50 年的计算机软件创作中一切顺利。
