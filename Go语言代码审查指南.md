[译]Go语言代码审查指南
===

- [Gofmt](#Gofmt)
- [Comment Sentences](#Comment-Sentences-注释语句)
- [Contexts](#Contexts-上下文)
- [Coping](#Coping-复制)
- [Crypto Rand](#Crypto-Rand)
- [Declaring Empty Slices](#Declaring-Empty-Slices-声明空切片)
- [Doc Comments](#Doc-Comments-文档注释)
- [Don't Panic](#Don't-Panic-不要使用\`panic\`)
- [Error Strings](#Error-Strings-错误信息)
- [Examples](#Examples-示例)
- [Goroutine Lifetimes](#Goroutine-Lifetimes-Goroutine-生命周期)
- [Handle Errors](#Handle-Errors-错误处理)
- [Imports](#Imports-导入包)
- [Import Blank](#Import-Blank-导入空)
- [Import Dot](#Import-Dot-导入点)
- [In-Band Errors](#In-Band-Errors)
- [Indent Error Flow](#Indent-Error-Flow)
- [Initialisms](#Initialisms-首字母缩写)
- [Interfaces](#Interfaces-接口)
- [Line Length](#Line-Length-行长度)
- [Mixed Caps](#Mixed-Caps-混合大小写)
- [Named Result Parameters](#Named-Result-Parameters-结果参数命名)
- [Naked Returns](#Naked-Returns-裸返回)
- [Package Comments](#Package-Comments-包注释)
- [Package Names](#Package-Names-包命名)
- [Pass Values](#Pass-Values-传值)
- [Receiver Names](#Receiver-Names-接受器名称)
- [Receiver Type](#Receiver-Type-接收器类型)
- [Synchronous Functions](#Synchronous-Functions-同步函数)
- [Useful Test Failures](#Useful-Test-Failures-有用的失败测试)
- [Variable Names](#Variable-Names-变量名称)

原文地址 https://github.com/golang/go/wiki/CodeReviewComments
更新时间2020-09-17

本文档收集了Go语言审查过程中一些常见的意见，整理出来的详细说明可以作为一个速记引用。这只是一个常见错误的清单，并不是一个全面的指导说明。

你可以把该文档作为 [Effective Go](https://golang.org/doc/effective_go.html)的补充说明。

## Gofmt
在你的代码上运行 [gofmt](https://golang.org/cmd/gofmt/)可以自动修复大部分机械样式的问题。几乎所有的Go代码提交前都使用`gofmt`进行格式化。文档剩余部分主要针对非机械样式的特点。

另一种可替换的方式是使用 [goimports](https://godoc.org/golang.org/x/tools/cmd/goimports)，这是一个`gofmg`的超集，可以在需要的时候自动添加（或删除）导入包的代码行

## Comment Sentences （注释语句）
参考 (https://golang.org/doc/effective_go.html#commentary) 。注释文档应该使用完整的句子，即便这样看上去有一点冗余。这种方法会使注释被提取到`godoc`文档中时获得更好的格式化。

注释应该以被描述对象的名称开头，并且以句号结尾：
```golang
// Request represents a request to run a command.
type Request struct { ...

// Encode writes the JSON encoding of req to w.
func Encode(w io.Writer, req *Request) { ...
```
等等。

## Contexts （上下文）
`context.Context`类型的值带有跨越API和进程边界的安全凭证，跟踪信息，截止时间和取消信号<sup>[1](#mfn1)</sup>。从传入RPCs和HTTP请求开始，一直到请求输出，Go程序在整个函数调用链中都显式传递Contexts。

大多数使用到Context的函数，都应当把Context作为函数的第一个参数，如：
```golang
func F(ctx context.Context, /* other arguments */) {}
```
如果一个函数还没有特定请求Context，可以使用`context.Background()`获取，但需要将err和Context同时传递，即使你认为你不需要。默认情况下只传递 Context；只有在你有充分的理由认为这是错误的，才能直接使用context.Background()。<sup>[2](#mfn2)</sup>

不要将Context成员添加到结构体类型中，而是应该将ctx参数添加到该类型需要传递的每一个方法上。一个例外是方法签名必须与标准库或第三方库中接口定义相匹配。

不要创建自定义Context类型，或者使用Context方法签名之外的接口。

如果需要传递应用数据，请将其放到参数、方法接收器和全局变量中，如果它确实属于Context，才放到Context的value属性中。

`Contexts`是不可变的，所以可以将相同的ctx传递给共享相同截止时间，取消信号，凭证，父级跟踪等的多个调用。

## Coping （复制）
为避免意外的别名，从另一个包中复制struct时候要小心。例如：`bytes.Buffer`包含一个`[]byte`切片，如果你复制一个`Buffer`，副本中的切片可能对原始数组进行别名操作，从而导致后续方法调用产生意想不到的效果。

通常，如果T类型的方法与其指针类型*T相关联，不要复制T类型的值。

## Crypto Rand
不要使用`math/rand`包来生成密钥，即便是一次性的密钥。在无种子的情况下，生成器是完全可以被预测的。即使使用`time.Nanoseconds()`作为种子，也只有几位的熵。请使用`crypto/rand.Reader`作为替代，如果需要文本格式，可以用十六进制或者base64编码打印：
```golang
import (
	"crypto/rand"
	// "encoding/base64"
	// "encoding/hex"
	"fmt"
)

func Key() string {
	buf := make([]byte, 16)
	_, err := rand.Read(buf)
	if err != nil {
		panic(err)  // out of randomness, should never happen
	}
	return fmt.Sprintf("%x", buf)
	// or hex.EncodeToString(buf)
	// or base64.StdEncoding.EncodeToString(buf)
}
```

## Declaring Empty Slices （声明空切片）
当声明一个空切片的时候，倾向使用
```golang
var t []string
```
代替
```golang
t := []string{}
```
前者声明一个nil的切片值，而后者是一个non-nil长度为0的切片。两者功能相同，长度和容量都是0，但是nil slice 是首选的风格。

请注意，在少数情况下，应选择non-nil长度为0的切片，例如编码JSON对象时（nil切片会被编码为null，而一个`[]string{}`将被编码为JSON空数组`[]`）。

设计接口时，请避免区分nil切片和non-nil零长度切片，这可能会造成一些细微的程序错误。

更多关于Go语言中nil的使用，请参考Francesc Campoy's的演讲 [Understanding Nil](https://www.youtube.com/watch?v=ynoY2xz-F8s).

## Doc Comments （文档注释）
所有顶级导出的名称都应该有文档注释，重要的未导出类型或函数声明也应如此。有关注释约定的更多信息，请参考(https://golang.org/doc/effective_go.html#commentary)

## Don't Panic （不要使用`panic`）
请参阅(https://golang.org/doc/effective_go.html#errors) 。不要使用`panic`作为一般错误的处理，请使用error和多返回值的方式。

## Error Strings （错误信息）
错误字符串不应该以大写字母开头（除非以专有名词或首字母缩写开头）或者以标点符号结尾，因为错误信息通常会跟在其他上下文之后打印输出。所以，应该使用`fmt.Errorf("something bad")`而不是`fmt.Errorf("Something bad")`，这样`log.Printf("Reading %s: %v", filename, err)`这样的格式中不会有额外的大写字母出现。这并不适用于日志记录，因为日志是隐式面向行的，而不是用于组合到其它信息中。

## Examples （示例）
当添加新的程序包时，请包含预期用途的示例：一个可运行的示例或包括完整调用链的简单测试示范。

获取更多信息请参阅 [testable Example() functions](https://blog.golang.org/examples)

## Goroutine Lifetimes （Goroutine 生命周期）
当你生成协程时，要清楚它们何时或是否会退出。

通过阻塞channel的发送和接收可能会引起协程的内存泄漏：即使通道被阻塞或无法访问，垃圾回收器也无法终止协程。

即使协程没有泄漏，当不再需要协程后仍然将其留在内存中也会造成其它一些微妙的难以诊断的问题。向已经关闭的通道发送信息会引发panic。在“不再需要结果之后”修改任然在使用的输入，任然会造成数据竞争。并且将协程留在内存中任意长时间会导致不可预测的内存使用。

请尽量保持并发代码足够简单，从而更容易的确认协程的生命周期。如果这不可行，请用文档说明协程何时以及为什么会退出。

## Handle Errors （错误处理）
请参阅(https://golang.org/doc/effective_go.html#errors) ，不要使用`_`变量丢弃错误。如果函数返回一个错误，请检查它以保证函数执行成功。处理函数返回的错误，将其返回或者在真正异常的情况下抛出panic。

## Imports （导入包）
避免重命名导入，除非为了避免包名冲突；好的包命名应该不需要重命名。发生冲突时，最好重命名本地或特殊项目的导入。

采用分组方式管理导入，用空行隔开各个分组。标准库的包应该总是在第一个分组。
```golang
package main

import (
	"fmt"
	"hash/adler32"
	"os"

	"appengine/foo"
	"appengine/user"

	"github.com/foo/bar"
	"rsc.io/goversion/version"
)
```
[goimports](https://godoc.org/golang.org/x/tools/cmd/goimports)会帮你做这件事。

## Import Blank （导入“空”）<sup>[4](#mfn4)</sup>
仅出于使用包的副作用而导入一个包时（使用语法`import _ "pkg"`）<sup>[3](#mfn3)</sup>，应仅在程序的main包或者需要它们的测试中导入。

## Import Dot （导入“点”）
部分包由于循环依赖，不能作为测试包的一部分进行测试，使用`import .`形式导入很有用，
```golang
package foo_test

import (
	"bar/testutil" // also imports "foo"
	. "foo"
)
```
在这个情况下，测试文件不能位于包`foo`中，因为它使用的`bar/testutil`依赖与`foo`包。因此我们使用`import .`形式以便测试文件伪装为`foo`包的一部分，即使它不是。除了这种情况之外，不要在程序中使用`import .`形式。由于不清楚一个像Quux这样的名称是一个当前包还是导入包的顶级标识符，这将使程序更加难以阅读。

## In-Band Errors
在C和类C的语言中，通常使函数返回类似-1或null的值来表示错误信号或缺少结果：
```
// Lookup returns the value for key or "" if there is no mapping for key.
func Lookup(key string) string

// Failing to check a for an in-band error value can lead to bugs:
Parse(Lookup(key))  // returns "parse failure for value" instead of "no value for key"
```
Go支持多返回值，提供了一种更好的解决方案。函数应该返回一个附加值以表示其他返回值是否有效，而不是要求客户端检查in-band 错误值。这个返回值可以是一个error，或者在不需要解释的时候可以是一个布尔值。它应该是最后一个返回值。
```golang
// Lookup returns the value for key or ok=false if there is no mapping for key.
func Lookup(key string) (value string, ok bool)
```
这样可以防止调用方错误的使用返回结果：
```golang
Parse(Lookup(key))  // compile-time error
```
并且鼓励编写跟具鲁棒性和可读性的代码：
```golang
value, ok := Lookup(key)
if !ok {
	return fmt.Errorf("no value for %q", key)
}
return Parse(value)
```
这条规则适用于导出的函数，对于未导出的函数也很有用。

当一个函数返回有效结果，即调用方不需要对结果做区别处理时，返回类似`nil`、`""`、`0`和`-1`的结果是没有问题的，

在一些标准库函数中（如`strings`包中的函数）会返回in-band错误值，这大大的简化了字符串处理的代码，代价是程序员需要做更多的工作。通常来说，Go代码应该返回表示错误信息的附加值。

## Indent Error Flow
尝试将代码路径保持在最小缩进，优先处理错误并缩进。允许快速视觉扫描常规路径可有效提升代码的可读新。例如，不要写：
```golang
if err != nil {
	// error handling
} else {
	// normal code
}
```
而应该写：
```golang
if err != nil {
	// error handling
	return // or continue, etc.
}
// normal code
```
如果if语句包括初始化语句，类似：
```golang
if x, err := f(); err != nil {
	// error handling
	return
} else {
	// use x
}
```
那么可能需要将短变量声明移动到新行：
```golang
x, err := f()
if err != nil {
	// error handling
	return
}
// use x
```

## Initialisms （首字母缩写）
名称中的缩写词或首字母缩写词（例如："URL" 或 "NATO"）保持一致的大小写。例如，"URL"应该显示为"URL" 或 "url" (比如 "urlPony", 或 "URLPony")，而不应使用"Url"。再例如：应使用ServeHTTP 而不是 ServeHttp。对于多个已初始化的“缩写词”，请使用"xmlHTTPRequest" 或 "XMLHTTPRequest"。

本规则同样适用于"ID"作为"identifier"缩写的情况<sup>[5](#mfn5)</sup>，所以应该用"appID" 而不是 "appId"。

使用`protocol buffer`编译器生成的代码不在本规则限定内。人工编写的代码应比机器编写的代码有更高的标准。

## Interfaces （接口）
Go接口通常属于使用interface类型值的包，而不是实现这些值的包。实现包应该返回具体的类型（通常是指针或者结果体）：这样，可以将新方法添加到实现中而不需要大量的重构。
```golang
package consumer  // consumer.go

type Thinger interface { Thing() bool }

func Foo(t Thinger) string { … }
```
```golang
package consumer // consumer_test.go

type fakeThinger struct{ … }
func (t fakeThinger) Thing() bool { … }
…
if Foo(fakeThinger{…}) == "x" { … }
```
不要在“用于模拟”的API实现端定义接口，而应该设计API，以便它可以使用真实实现的公开API对其进行测试。
```golang
// DO NOT DO IT!!!
package producer

type Thinger interface { Thing() bool }

type defaultThinger struct{ … }
func (t defaultThinger) Thing() bool { … }

func NewThinger() Thinger { return defaultThinger{ … } }
```
相反，返回一个具体的类型，让消费者模拟生产者实现。
```golang
package producer

type Thinger struct{ … }
func (t Thinger) Thing() bool { … }

func NewThinger() Thinger { return Thinger{ … } }
```
在使用接口之前不要定义接口：没有一个实际的用法示例，很难判断一个接口是否是必需的，更不用说它应该包括什么方法了。

## Line Length （行长度）
Go代码没有对于代码行长度的硬性限制，但是应避免影响阅读的长行。同样的，如果长行可读性更好，不要为了缩短行而添加换行符——例如，行组成是重复的。

大多数情况，当人们“不自然的”换行（比如在函数声明或函数调用的中间，尽管有一些例外），如果有合理的参数数量和合理简短的变量名，那么换行就完全没有必要。看起来长行似乎与长名称有关，避免名称过长有很大帮助。

换句话说，换行是因为你所写内容的语义（作为基本规则）而不是行的长度。如果你发现行太长，改变名称或语义可能会得到不错的结果。

实际上，这是一个和函数应该多长完全相同的建议。没有这样一个规则说“绝不应该有超过N行的函数”，但是肯定存在类似函数太长、太零碎而不连贯这样的问题，解决方法是改变函数的边界位置，而不是执着于行数。

## Mixed Caps (混合大小写)
请参阅 (https://golang.org/doc/effective_go.html#mixed-caps) 。即使违反了其他语言的约定也应如此。例如一个未导出的常量应该命名为 maxLength 而不是 MaxLength 或 MAX_LENGTH

另见[Initialisms](#Initialisms)

## Named Result Parameters （结果参数命名）
考虑一下如下面为结果参数命名后的代码，在 godoc 中看起来是什么样的：
```golang
func (n *Node) Parent1() (node *Node) {}
func (n *Node) Parent2() (node *Node, err error) {}
```
在 godoc 中会显得很不连贯；所以最好用这样的方式：
```golang
func (n *Node) Parent1() *Node {}
func (n *Node) Parent2() (*Node, error) {}
```
另一方面，如果一个函数返回两、三个同种类型的参数，或者如果根据上下文不能清晰的表明返回结果的意思，那么在某些上下文中添加命名可能是很有用的。
```golang
func (f *Foo) Location() (float64, float64, error)
```
上面的代码就不如下面的清晰：
```golang
// Location returns f's latitude and longitude.
// Negative values mean south and west, respectively.
func (f *Foo) Location() (lat, long float64, err error)
```
但是不要仅仅为了不想在函数内声明变量而做结果参数命名；这样做是以不必要的API冗长性为代价，换取一点微小的简洁实现。

如果函数只有几行，那么使用[裸返回](#Naked Returns)是可以的。一旦函数增加到中等规模，请明确说明你的函数返回值。
推论：仅仅因为可以使用裸返回而命名结果参数是不值得的。清晰的文档总比在函数中少写一两行代码要重要。

最后，在有些情况下你需要命名结果参数才可以在延迟闭包中对其进行修改，那么这也是可以的。

## Naked Returns (裸返回)
指从某个函数返回时，没有明确说明返回内容的return，直接使用命名结果参数的变量返回。
参阅 [Named Result Parameters](#Named Result Parameters)。

## Package Comments （包注释）
与godoc展现的所有注释一样，包注释应该出现在包声明的前面，没有空行。
```golang
// Package math provides basic constants and mathematical functions.
package math
```
```golang
/*
Package template implements data-driven templates for generating textual
output such as HTML.
....
*/
package template
```
对于“main包(package main)”注释，在二进制名称之后可以使用其他样式的注释（如果它们放在前面，则可以大写），例如，对于`seedgen`目录下的main包注释，你可以写成这样：
```golang
// Binary seedgen ...
package main
```
或是
```golang
// Command seedgen ...
package main
···
或是
```golang
// Program seedgen ...
package main
```
或是
```golang
// The seedgen command ...
package main
```
或是
```golang
// The seedgen program ...
package main
```
或是
```golang
// Seedgen ..
package main
```
以上是相应示例，它们的合理变体也是可以接受的。

请注意，以小写字母单词开头的句子不属于包注释的可接受选项，因为它们是公开可见的，应该使用合适的英文书写包括句子的第一个单词使用大写字母。当二进制的名称是第一个单词时，即使它与命令行调用的拼写不完全匹配也必须将其首字母大写。
关于注释约定的更多信息，请参阅(https://golang.org/doc/effective_go.html#commentary)

## Package Names （包命名）
包中对名称的所有引用都将用包名来完成，所以你可以从标识符中省略该名称。例如，如果有一个包名为`chubby`，你不需要将类型命名为`ChubbyFile`，这样调用方需要写成`chubby.ChubbyFile`，而是应该将类型命名为`File`，这样调用方代码可以写成`chubby.File`。避免使用无意义的包名，如 util、common、misc、api、types、interfaces等。
更多信息，请参阅(http://golang.org/doc/effective_go.html#package-names) 和 (http://blog.golang.org/package-names)

## Pass Values （传值）
不要仅为了节省几个字节把指针作为函数参数。如果一个函数在整个过程中只引用它的参数x作为*x，那么这个参数不应该是一个指针。常见的实例包括传递给`string(*string)`的指针或指向结果值`(*io.Reader)`的指针。在这两种情况下，值本身都是固定大小，可以直接传递。这个建议不适用于大型结构体，或者是现在还很小但是以后可能会变大的结构体。


## Receiver Names （接受器名称）
方法接收器的名称应反映其身份，通常，使用其类型的一到两个字母缩写就足够了（如：用"c" 或 "cl" 表示 "Client"）。不要使用通用名称，如："me", "this"或"self"等，这是面向对象语言的典型标识符，会赋予方法特殊的含义。在Go语言中，方法的接收器只是另一个参数，因此应相应的命名。该名称不必像方法参数那样具有描述性，因为它的作用是显而易见的，并且不提供任何文档目的。名称可以非常短，因为它将出现在该类型每个方法的几乎每一行上<sup>[6](#mfn6)</sup>。也应当保持一致，如果在一个方法的接收器称为"c"，则在其它方法中就不要用"cl"。

## Receiver Type （接收器类型）
方法应该使用值接收器还是指针接收器是很难选择的，特别是对于Go新手程序员。如果有疑问，那就使用指针。但是有的时候出于效率的考虑，使用值接收器是合理的，例如小的不变的结构体或基础数据类型。以下是一些有用的指导：
- 如果接收器是`map`、`func`或`chan`类型，请不要使用指向他们的指针。如果接收器是一个`slice`，并且该方法不会重新切片或重新分配切片，不要使用指向该切片的指针。
- 如果方法需要修改接收器，则接收器必须是指针。
- 如果接收器是一个包含`sync.Mutex`或类似同步字段的结构体，则接收器必须是指针以避免复制。
- 如果接收器是一个大型结构体或数组，那么使用指针接收器更有效。多大算大？假设它相当于将其所有元素作为参数传递给方法，如果感觉太大了，那么对接收器来说就是太大了。
- 函数或方法可以改变接收器吗（并发调用或被当前方法调用时）？当方法被调用时，会创建一个接收器的值类型副本，所以外部更新不会影响到此接收器。如果必须在原始接收器中看到变更，接收器必须是一个指针。
- 如果接收器是结构体，数组或切片，并且其任何元素是指向可能被修改的对象的指针，最好使用指针接收器，这样能使意图更容易被读者了解。
- 如果接收器是小型结构体或数组，那么它自然是一个值类型（例如类似`time.Time`类型），没有可变字段且没有指针，或只是一个简单的基础数据类型（类似`int`或`string`），则使用值接收器会更合理。使用值接收器可以减少可能生成的垃圾数量，如果将值类型传递给值方法，则可以使用栈上的副本代替在堆上进行分配（编译器会尝试避免这种分配，但不一定总能成功）。因此，在没有进行分析之前，不要选择值接收器类型。
- 最后，如有疑问，请使用指针接收器。

## Synchronous Functions （同步函数）
优先推荐同步函数而不是异步函数。同步函数会在返回之前直接返回结果、结束所有回调和通道操作。

同步函数使goroutines在调用中保持本地化，这样更容易推断其生命周期以避免内存泄漏和数据竞争。同步函数也更容易测试：调用者可以传递输入并检查输出，而不需要处理拉取和同步。

如果调用者需要更多的并发，可以很容易的通过从单独的goroutine中调用函数来实现。但在调用端移除不必要的并发性是非常困难的，有时候是不可能的。

## Useful Test Failures （有用的失败测试）
失败的尝试应该提供有用的信息，说明错误是什么，输入内容是什么，期望获得什么以及实际得到什么。编写一堆 assertFoo 帮助程序可能很吸引人，但是请确保你的帮助程序能产生有用的错误信息。假设调试失败测试的人不是你，也不是你的团队。典型的Go失败测试如下：
```golang
if got != tt.want {
	t.Errorf("Foo(%q) = %d; want %d", tt.in, got, tt.want) // or Fatalf, if test can't test anything more past this point
}
```
请注意这个地方的顺序`actual != expected`，并且错误消息也使用相同的顺序。有的测试框架鼓励使用backwards的写法：`0 != x`，“期望为0，实际得到 x”，等等。但Go不是这样的。

如果看起来需要写很多东西，你可能需要写一个表驱动测试。

当使用带有不同输入的测试帮助程序时，另一个常见消除失败测试歧义的技术是为每一个调用者封装不同的TestFoo函数，而测试名称也根据对应的输入命名：
```golang
func TestSingleValue(t *testing.T) { testHelper(t, []int{80}) }
func TestNoValues(t *testing.T)    { testHelper(t, []int{}) }
```
在任何情况下，你有责任为将来任何可能调试你的代码的人提供有用的信息。

## Variable Names （变量名称）
Go语言中的变量名应该尽量简短，对于范围有效的局部变量尤其如此。例如用 c 代替 lineCount，用 i 代替 sliceIndex。

基本规则是：使用变量的地方离声明越远，命名应越具有描述性。对于方法的接收器，一两个字母就足够了，诸如循环索引和读取器之类的公共变量可以是单个字母(i, r)。特别的事物以及全局变量则需要更具描述性的名称。

# 说明
<a name="mfn1">1</a>: 参考代码注释 (https://github.com/golang/go/blob/master/src/context/context.go)

Package context defines the Context type, which carries deadlines, cancellation signals, and other request-scoped values across API boundaries and between processes.

<a name="mfn3">2</a>: 此处翻译不确定，原文如下：

A function that is never request-specific may use context.Background(), but err on the side of passing a Context even if you think you don't need to. The default case is to pass a Context; only use context.Background() directly if you have a good reason why the alternative is a mistake.

结合代码注释以及官方博文(https://blog.golang.org/context)讨论，推断 Background 方法会返回一个 non-nil 的空 Context，通常用在主函数、初始化、测试等地方，作为请求开始的根 Context；因此除了这些起始位置，一般不建议在中间代码使用Background()。

<a name="mfn3">3</a>: 当导入一个包时，该包下的文件里所有init()函数都会被执行，然而，有些时候我们并不需要把整个包都导入进来，仅仅是是希望它执行init()函数而已。这个时候就可以使用 import _ 引用该包。即使用`import _ "pkg"`只是引用该包，仅仅是为了调用init()函数，所以无法通过包名来调用包中的其他函数。

<a name="mfn4">4</a>: 不是特别确定此处In-Band的含义，个人推测In-Band表示返回值中混合了表示错误信息的值，需要调用方区别处理，而Out-Band

<a name="mfn5">5</a>: 此处原文有补充说明，翻译不确定，绝大多数情况下"id"不是用以指代“本我”和“超我”

which is pretty much all cases when it's not the "id" as in "ego", "superego"

<a name="mfn6">6</a>: 不确定含义，原文 familiarity admits brevity
