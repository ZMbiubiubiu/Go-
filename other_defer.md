
- [defer 的定义](#defer的定义)
- [为什么要有 defer](#为什么要有defer)
- [defer 的特性](#defer的特性)
- [如何记住 defer 的特性](#如何记住defer的特性)


# defer 的定义
> A defer statement defers the execution of a function until the surrounding function returns.

一句话总结：`defer`就是用来延迟执行函数的技术手段。

虽然只有短短一句话，但是我们也可以从中获取很多信息：

1、`defer`后面跟函数:`defers the execution of a `**`function`**（和方法`method`，后文会讲到方法`method`本质上就是函数`function`）。

2、`defer`的作用就是在特定时机激活后接的函数:`the execution`。

3、所谓的特定时机:`until the surrounding function returns`。

我们会发现，下面这句话就是对上述三条总结的再次说明。

> A "defer" statement invokes a function whose execution is deferred to the moment the surrounding function returns, either because the surrounding function executed a return statement, reached the end of its function body, or because the corresponding goroutine is panicking.

# 为什么要有 defer
再次重申，**`defer`的本质就是延迟自动执行函数**。

它的出现就是为了更加便利的进行资源管理，减轻程序员的心智负担，提高代码的可读性、可维护性和可扩展性。

假如`Go`没有`defer`机制，为了实现一个锁保护的文件操作。我们得这么写：

```Go
func writeToFile(fname string, data []byte, mu *sync.Mutex) error {
    mu.Lock()
    f, err := os.OpenFile(fname, os.O_RDWR, 0666)
    if err != nil {
    mu.Unlock()
    return err
    }

    _, err = f.Seek(0, 2)
    if err != nil {
        f.Close()
        mu.Unlock()
        return err
    }

    _, err = f.Write(data)
    if err != nil {
        f.Close()
        mu.Unlock()
        return err
    }

    err = f.Sync()
    if err != nil {
        f.Close()
        mu.Unlock()
        return err
    }

    err = f.Close()
    if err != nil {
        mu.Unlock()
        return err
    }

    mu.Unlock()
    return nil
}
```

就像老奶奶的裹脚布——又臭又长。

我们每次操作，都得处理资源的释放：锁的释放、文件句柄的关闭。

仅仅两个资源的释放操作，我们在编写代码就需要『如临深渊，如履薄冰』。增加资源的参与数，那简直不堪设想。

幸好，`Go`有`defer`机制，我们可以改写如下：

```go
func writeToFile(fname string, data []byte, mu *sync.Mutex) error {

    mu.Lock()
    defer mu.Unlock()

    f, err := os.OpenFile(fname, os.O_RDWR, 0666)
    if err != nil {
        return err
    }
    defer f.Close()
    
    _, err = f.Seek(0, 2)
    if err != nil {
        return err
    }

    _, err = f.Write(data)
    if err != nil {
        return err
    }

    return f.Sync()
}
```

我们所需要做的就是申请资源之后，马上`defer+`释放资源的逻辑。当`writeToFile`运行结束时，会自动释放锁和关闭文件句柄。

# defer 的特性

上面对`defer`的描述，只是非常粗糙的讲解。下面让我们沉浸到`defer`的具体技术细节。

为了讨论一个技术，提出好的问题是理解其要领的因素之一。比如：

* `defer`后接函数，那么函数的参数什么时候确定？（要知道同样的变量，在函数退出时与刚进入函数时，内容可能是不一样的）
* 可以后接方法吗？

* `defer`后的函数真正执行时机是：声明`defer`表达式的函数执行结束时。那么这个所谓的函数执行结束时，是返回后，还是返回前？还是其他什么时间点？
* 如果一个函数之中，声明了多个`defer`表达式，虽然都是函数退出时调用，但是它们执行的顺序如何？随机的？（像`select channel`一样）？先进先出？（越先被`defer`的函数越早调用）？还是先进后出？

没错，有很多问题会萦绕心头。来吧，让我们回答上面的问题。

不过在此之前，学习两个英文单词。

`execution`：中文是执行的意思。比如函数的执行，就是运行函数的意思。

`evaluate`：中文是评价、估计的意思。在编程里面，比如函数参数，当我们说`evaluate function parameters`，就是确定函数参数值的意思。嗯，`evaluate`可以翻译成「求值」。

下面进入正题：

**1、defered function evaluates its parameter when is occur**

被`defer`的函数会立马求值函数的参数

```go
func print(num int) {
    fmt.Println(num)
}

func foo() {
    var i = 666
    defer print(i)
    
    i = 888
    return
}

func main() {

    foo()
}
```

会打印`666`而不是`888`。

这就叫做当出现`defer`时，后接函数的参数立马被求值，继而确定，即`i=666`。

虽然`deferred function`在后面才会执行，但是参数从一开始就确定了。

**2、defered function will execute before real return**
被`defer`的函数会在调用它的函数返回前执行

这句话不好理解。重点在于什么叫做`real return`。

休息一下，下面的例子需要仔细看：

```go
func sara() (result int) {

    result = 1

    defer func() {
        result = 666
    }()

    return result
}

func lisa() int {

    var result = 1
    
    defer func() {
        result = 666
    }()

    return result
}

func main() {

    r := sara()
    fmt.Printf("sara finially result, %d\n", r) // 666

    r = lisa()
    fmt.Printf("lisa finially result, %d\n", r) // 1
}
```

可以观察到，`sara`和`lisa`基本逻辑是一样的，唯一的不同就是`sara`是具名返回值函数，而`lisa`是匿名返回值函数。而这一点区别可太重要了，直接导致不同的处理逻辑。

好好理解一下上面的例子，下面给出`Go`函数的返回值模型

其实`go`的`return`语句是分成两部分的。比如对于匿名返回值函数`lisa`来说

```go
return result
相当于

1、var ret = result // ret 是真返回值
2、return ret
```

对于具名返回值函数`sara`来说

```go
return result
相当于

1、var result = result
2、return result
```

注意两者的区别。

重点来啦，`defer`的运行时机就是步骤`1`和步骤`2`之间！

* 对于普通函数`lisa`来说，我要返回的是`ret`，就算你在`defer`中修改了`result`，也不会影响到最终返回值哦。
* 对于具名返回值函数`sara`就不一样了，她返回的就是`result`，在`defer`中修改了`result`，最终的结果就不一样啦。

咳咳，具名返回值和`defer`的联合使用，要注意哦😑。别写这样的代码。

**3、多个`defer`的执行的顺序是：后进先出，类似栈**
```golang
func print(num int) {
    fmt.Println(num)
}

func bob() {
    defer print(1)
    defer print(2)
    defer print(3)

    return
}

func main() {
    bob()
}
```

会打印

```text
3
2
1
```

# 如何记住 defer 的特性

`defer`出现就是为了资源的管理。

比如有以下这个过程：首先执行某个操作，需要首先获取资源`A`，然后获取资源`B`，最后获取资源`C`。那么`defer`的顺序，肯定是先放回`C`资源，然后`B`资源，最后是`A`资源。

这个过程让我想起线性代数中矩阵的乘法和逆运算——逆矩阵。

比如`AB`，表示矩阵`A`乘以矩阵`B`，那么它俩的逆运算呢？

$(AB)^{-1} = B^{-1}A^{-1}$

当时为了记住这个特性，我是记住了下面的这个类比：

先穿袜子，然后穿鞋子；如果这一过程反过来，那就是先拖鞋，再脱袜子。

`nice`。

你不能先脱袜子，再拖鞋啊！

如今看来，不论是穿衣服、矩阵乘法、资源的管理，都是一种行为、一种运动、一种变化。他们是分先后的，所以他们的“反运动”要与正向运动相反。
