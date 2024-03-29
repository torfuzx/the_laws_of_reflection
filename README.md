反射的法则
=========

注：本文是[The Laws of Reflection](http://blog.golang.org/laws-of-reflection)的中文翻译，转载请注明出处。


简介
----

反射在计算机领域指的是程序检查自身结构的能力，这通常是通过类型来完成的。这是一种元编程的形式。这同时也是困惑的源头之一。

在这边文章中，我们试着通过解释在go中反射是如何工作的让你理解反射。每一种语言的反射模型都是不同的（并且很多语言压根不支持反射），但是这篇文章是关于Go的，因此文章接下来的部分，请把“反射”反射当做“Go中的反射”来理解。


类型和接口
---------

因为反射简历在类型系统之上，让我们首先从Go中的类型讲起。

Go是静态语言。每一个变量都有一个静态的类型，即在编译的时候确切的已知和确定的一种类型：int，float32,*MyType，[]byte等等。如果我们声明：

```
type MyInt int

var i int
var j MyInt
```

那么i的类型为int，j的类型为MyInt。变量i和j拥有截然不同的两种类型的静态类型，尽管底层的类型是一样的，你不能把一种变量类型的值赋给另外一个变量，要赋值必须通过类型转换。

一类重要的类型的就是接口类型，它们代表着一系列方法。一个接口变量可以存储任何具体（非接口）的值，只要这些值实现了接口的方法。一个常见的列子就是io.Reader和io.Writer，Reader和Writer两种类型来源与io包。

```
// Reader是一个接口，包含着基本的Read方法
type Reader interface {
    Read(p []byte) (n int, err error)
}

// Writer是一个接口，包含着基本的Write方法
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

任何类型实现了Read（或者Write）方法且签名一样就被称作实现了io.Reader（或者io.Writer）接口。在下面的讨论中，意味着一个io.Reader类型的变量可以保存任何有一个Read方法的值。

```
var r io.Reader
r = os.Stdin
r = bufio.NewReader(r)
r = new(bytes.Buffer)
// and so on
```

十分重要的是，不论任何变量r可能保存任何具体的值。r的类型永远是io.Reader：Go是静态类型语言，r的静态类型是io.Reader。

一个相当重要的接口类型的例子是空接口：

```
interface{}
```

他代表着一个空的方法集，并且任何的值都满足空接口，因为任意的值都有零个或者更多的方法。

有的人说Go的接口是动态类型，但是这么说事误导人的。接口是静态类型：一个接口类型的变量永远有一样的的静态类型，即使在运行的时候，保存在接口变量中的实际值的类型可能会改变，保存的值总是满足该接口的。

我们需要对此认识十分清楚，因为反射和接口是紧密结合的。


反射的表现
---------

Russ Cox写了一篇详细的博客介绍Go中接口值是如何表现的。完全的重复该篇文章里的内容是没有必要的，但是简单的的总结却是可以的。

一个接口的变量是按对保存的：一个赋给该变量具体的值，和该值的类型描述符。更加确切的说，值是底层实现该接口的值和完整描述该条目的的类型描述。例如：

```
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```

在这之后，r从形式上包含了一个(value，type)对，（tty，*os.File）注意类型*os.File不止实现了Read方法；尽管接口值只提供访问Read方法，内部的值携带着所有与该值的类型信息。因此我们可以做如下操作：

```
var w io.Writer
w = r.(io.Writer)
```

这个语句中的表达式是一个类型的断言，其断言的内容是变量r内部保存的内容同时也实现了io.Writer接口，因此我们可以把她赋值给w。赋值之后，w会包含一个对（tty, *os.File）。这个值-类型对于保存在r中的值-类型对是一样的。接口的静态类型决定了在接口类型的变量可以调用什么方法，尽管该变量内含的具体值可能会有一个更大的方法集。

我们还可以做如下操作：

```
var empty interface{}
empty = w
```

这样空接口的值会再次包含同样的类型-值对（tty，*os.File）。这十分的实用：一个空接口能保存任何的的值，并且包含所有我们可能需要一个值的信息。


此处我们不需要做类型断言因为静态上w满足了空接口。在上面的例子中，我们把值从一Reader接口移到了Writer接口，我们需要显式的使用类型断言因为Writer的方法集不是Reader方法集的一个子集。

一个重要的细节是一个接口里面的的对的形式是（值，具体的类型）而不是（值，接口的类型）。接口不保存接口的值。

现在我们可以开始讲反射了。

1. 反射的第一条定律 - 反射从接口的值到反射的对象
----------------------------------------
在最基本的层面，反射只是一个检查一个保存在接口变量中（类型，值）对的机制。首先，我们需要知道reflect包中的两个类型：Type和Value。这两个类型能让我们访问一个接口变量里面的内容。我们可以使用两个方法，reflect.TypeOf和reflect.ValueOf来获取一个接口变量中relefct.Type和reflect.Value的那一部分。（同时我们也能很容易的从reflect.Value中获取reflect.Type，但是让我们暂时Value和Type的概念分开。）

让我们从TypeOf开始：

```
package main

import (
    "fmt"
    "reflect"
)

func main() {
    var x float64 = 3.4
    fmt.Println("type:", reflect.TypeOf(x))
}
```

这个程序会输出：

```
type: float64
```

你可能会想这里的接口在那里。因为这个程序看上去像是在传递一个float64的变量x，而不是在一个接口的值到reflect.TypeOf方法。但是接口是存在的，如godoc中显示的，reflect.TypeOf的方法签名包含一个空的接口：

```
// TypeOf返回保存在接口interface{}中的值的类型
func TypeOf(i interface{}) Type
```

当我们调用reflect.TypeOf(x)的时候，x首先会被保存到一个空的接口口里面然后作为参数传递。reflect.TypeOf会解包该空接口信息并还原其类型信息。

reflect.ValueOf的方法当然能恢复接口中的值。

```
var x  float64 = 3.4
fmt.Println("value:", reflect.ValueOf(x))
```

会输出：

```
value: <float64 Value>
```

reflect.Type和reflect.Value都有很多方法让我们检查和操作他们。一个重要的例子就是Value有一个Type的方法可以返回一个reflect.Value的Type。另外一个是Type和Value都有一个Kind方法可以返回一个常量来表明保存的哪一种值：Uint，Float64，Slice等等。同时Value上的一些诸如Int和Float的方法可以让我们（以int64和float6）来获取保存在接口内部的值。

```
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())
fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
fmt.Println("value:", v.Float())
```

会输出：

```
type: float64
kind is float64: true
value: 3.4
```

同时也有一些诸如SetInt和SetFloat的方法，但是为了方便使用他们我需要了解可设置性（settability）,是下边要讨论的第三个法则。

reflection库中有一些属性值得单独拿出来讲。首先，为了保持API的简洁，Value的“getter”和“setter”方法操作于能保存值的最大类型：例如int64代表所有有符号型的整型。即，Value的Int方法返回一个int64，SetInt接收一个int64的参数。使用这些方法的时候可能需要做类型的变换。

```
var x uint8 = 'x'
v := reflect.ValueOf(x)
fmt.Println("type:", v.Type())                            // uint8.
fmt.Println("kind is uint8: ", v.Kind() == reflect.Uint8) // true.
x = uint8(v.Uint())
```
第二个特性是一个反射对象的Kind描述的时底层类型，而不是静态类型。如果一个反射对象包含一个用户定义的的整形类型，如在

```
type MyInt int
var x Myint = 7
v := reflect.ValueOf(x)
```

v的Kind仍然是reflect.Int，即使v的静态类型是MyInt而不是int。换句话说，Kind不能将一个int和MyInt区分开来，但是Type可以。


2. 反射的第二条法则 - 反射从反射对象到反射的值
----------------------------------------

和物理反射一样，Go中的反射会生成其自身的对立面。

拿到一个reflect.Value我们通过使用Interface方法可以恢复接口的值；该方法的实际效果是将类型和值的信息打包回接口的表现的形式并且返回结果：

```
// Interface以interface{}的形式返回v的值
func (v Value) Interface() interface{}
```

结果我们可以用：

```
y := v.Interface().(float64) // y将会拥有类型float64
fmt.Println(y)
```

来打印出反射对象v代表的float64值

然而，我们还可以更进一步。fmt.Println，fmt.Printf等等的参数都是以空接口值的形式传递的，然后种子fmt宝的内部对传入的值进行解包，正如我们在之前的例子中看到的一样。因此为了打印relect.Value的内容，需要传递Interface方法的结果到打印方法：

```
fmt.Println(v.Interface())
```

（为什么不是fmt.Println(v)? 因为v是一个reflect.Value；我们需要其保存的具体的值）因为我们的值是一个float64，我们甚至可以使用一个浮点数的格式：

```
fmt.Println("value is %7.1e\n, v.Interface())
```

并且得到如下的结果：

```
3.4e+000
```

同样的，这里没有必要对v.Interface()的结果做类型是float64的类型断言；空接口的值里面保存着值的类型信息，Printf可以对其进行恢复。

再次重申：反射从反射的值到反射的对象，并且可以从反射对象到反射的值。

3. 反射的第三法则 - 要修改反射对象，其值必须是可以设置的
----------------------------------------

第三条法则是最难以理解和困惑的，但是如果我们回到第一条法则就会变得容易的多。

这里是一些不能运行的代码，但是值得拿来探讨下：

```
var x float64 = 3.4
v := refelct.ValueOf(x)
v.SetFloat(7.1) // 错误，会panic
```

如果你运行这段代码，程序会panic并带着如下错误信息：

```
panic: reflect.Value.SetFloat using unaddressable value
```
问题不在于值7.1不可寻址，而是v不可设值。设值性是一个反射Value的属性，并不是所有的反射Value都有这个属性。

Value的CanSet方法能返回一个Value的设值性；在我们这个例子中：

```
var x float64 = 3.4
v := reflect.ValueOf(x)
fmt.Println("settability of v:", v.CanSet())
```

输出：

```
settability of v: false
```

在一个不可设置的Value上调用Set方法是会报错的。但是什么是设值性？

设值性有一点像可寻址性，但是更加的严格。他是反射对象的一种属性，表示可以修改用来创建反射对象的实际存储。设置性由是否反射对象保存在原始对象。

当我们说：

```
var x float64 = 3.4
v := reflect.ValueOf(x)
```

我们传递了一个xd的拷贝到reflect.ValueOf，因此作为reflect.ValueOf的参数创建的反射值，是一个x的拷贝值而不是x自身，因而，在语句中：

```
v.SetFloat(7.1)
```

如果这条语句能成功执行，其不会更新x，即使v看起来像其从x中创建的。相反，其会更新在反射值里x的一个拷贝而不是x自身不会受到影响。这样会招来更多的疑惑，也没有什么实际意义。因此这是非法的，而设置性是一个属性用来规避这个问题。

如果这看起来有点过了，其实不是。这是一种熟悉的情形。想象一下将x传递给一个函数：

```
f(x)
```

我们不会期待f真的能修改x的值因为我们传递的时x值的一个拷贝，而不是x本身。如果我们想f能修改直接修改x我们必须传递x一个地址，即指向x的指针：

```
f(&x)
```

这听起来十分容易理解和熟悉，反射跟这是一个原理。如果我们想通过反射改变x的值，我们必须给反射一个我们想修改值的指针。

让我们试试，首先我们照常初始化x并且创建一个指向其的反射值，让我们称为p。

```
var x float64 = 3.4
p := reflect.ValueOf(&x) // Note: take the address of x.
fmt.Println("type of p:", p.Type())
fmt.Println("settability of p:", p.CanSet())
```

现在的输出是：

```
type of p: *float64
settablity of p: false
```

反射对象不可设值，但是我们不是想设置p的值，而是*p。需要得到p指向的对象，我们调用Value上的Elem方法，其通过指针间接取值，并且将结果保存进一个反射Value，姑且称为v。

```
v := p.Elem()
fmt.Println("settability of v:", v.CanSet())
```

现在v是一个可设值的的对象，正如结果显示的一样：

```
settability of v: true
```

并且，由于其代表x，我们终于能通过v.SetFloat来改变x的值：

```
v.SetFloat(7.1)
fmt.Println(v.Interface())
fmt.Println(x)
```

其输出正如期待的一样，是：

```
7.1
7.1
```

反射可能会很难理解，但是其做的事情跟语言做的事情一样，除了通过反射Types和Values能掩盖下面发生的事情。只需要记住，反射的Value需要某事物的地址，才能够修改其代表的值。


结构
----

在我们之前的例子中v自身不是一个指针，其只是从指针派生的东西。一个常见的用法是使用反射来修改一个结构的的字段。只要我们拥有结构的地址，我们就能修改其字段的值。

这里有一个简单的例子来分析结果的值，t。我们使用结构的地址创建了一个反射对象因为我们之后想修改它。然后我们设置typeOfT并遍历字段使用直接的方法。注意我们抽出字段的名字，但是那些字段本身是通常的reflect.Value对象。

```
type T struct {
    A int
    B string
}
t := T{23, "skidoo"}
s := reflect.ValueOf(&t).Elem()
typeOfT := s.Type()
for i := 0; i < s.NumField(); i++ {
    f := s.Field(i)
    fmt.Printf("%d: %s %s = %v\n", i,
        typeOfT.Field(i).Name, f.Type(), f.Interface())
}
```

这个程序的输出是：

```
0: A int = 23
1: B string = skidoo
```

关于可设值性还有一个点：字段的名称是大写的，即导出的，因为只有导出的字段可以设置。

因为s包含一个可以设值的的反射对象，我们可以修改结构的的字段。

```
s.Field(0).SetInt(77)
s.Field(1).SetString("Sunset Strip")
fmt.Println("t is now", t)
```

这里是结果：

```
t is now {77 Sunset Strip}
```

如果我们修改程序从t中来创建s，而不是&t，调用SetInt和SetString就会失败，因为t的字段是不可设值的。

结尾
---

让我们总结下反射的法则：

- 反射从反射的值到反射的对象
- 反射从反射的对象到反射的值
- 要修改一个反射对象，其值必须是可设置的

一旦你理解了这些反射的法则，反射会变得很容易使用。尽管其仍然很难懂。反射是一个强大的工具，使用时需要小心，并且需要避免除非严格必要。

关于反射我们还有很多没有讲到 - 通过通道发送和接受，安置内存，使用切片和映射，调用发发和函数 - 但是这篇文章已经够长的了，我们将会在之后的文章中讲其他的一些主题。

By Rob Pike
