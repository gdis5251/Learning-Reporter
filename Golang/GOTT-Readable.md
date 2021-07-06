# Golang On The Toilet--Readable

> golang on the toilt 该名称源于「谷歌」的 google on the toilet，谷歌公司会在厕所里贴一些技术上的小知识点或注意点，让大家在上厕所的时候都可以学习一下。
>
> 所以本篇博客，也是诺列一些在平时开发中，使用 golang 的一些小知识点。

## 提升代码的可读性

总结一句话就是，**你写的代码，不要仅限于别人能看懂，而且还要轻松的看懂**。不要在交接工作或者同事要看你代码学习的时候，同事心里骂娘。要写一些简单并且可读性高的代码。不仅仅是为了别人，也为了自己。

### if, else and happy path

#### 1. 避免在 else 里 return

```go
// Bad example
func abs(x int) int {
  if x >= 0 {
    return x
  } else {
    return -x
  }
}

// Good example
func abs(x int) int {
  if x >= 0 {
    return x
  }
  
  return -x
}
```

- 在 Bad example 中，else 里也返回了一个值，其实这增加了理解成本，所以这里可以直接去掉 else，在最后返回就好。

#### 2. 尽早返回 error，消除 happy path

> happy path 简单的说就是：完全不考虑异常

```go
// Bad example 2
func aFunc() error {
    err := doSomething()
    if err == nil {
        err := doAnotherThing()
        if err == nil {
            return nil // happy path
        }
        return err
    }
    return err
}

// Good example 2
func aFunc() error {
    err := doSomething()
    if err != nil {
        return err
    }
    err := doAnotherThing()
    if err != nil {
        return err
    }
    return nil
}

```

- 在上述例子中，Bad example 接受一个 err，没有及时处理如果发生错误的情况，而是选择了没有错误的情况继续 do something，最后才处理错误，这样就会使代码可读性变差。
- 在 Good example 中，每一步都及时处理错误，显而易见，Good example 的代码可读性就高了很多。

#### 3. 除非必要，否则可以在初始化的时候赋值

```go
// Bad example 3
var a int
if flag {
    a = 1
} else {
    a = -1
}
// Good example 3
var a = -1
if flag {
    a = 1
}
```

- 在 Bad example 中，先声明一个变量，在通过 flag 判断两种情况发分别对变量进行赋值，这样首先不管可读性，个人感觉都非常麻烦。
- 如果有这种场景，可以直接使用 Good example ，直接在声明的时候对变量赋值。

#### 4. 除非必要，否则避免写 init() 函数

```go
package example

func init() {
  // do something aim to product
  // do something very heavy
}

func InitExample() {
  // do something aim to product
  // do something very heavy
}

func InitDebugExample() {
  // do something aim to product
  // do something very heavy
}
```

- 首先解释 init() 函数会被怎么调用，在 golang 里，只要是引用了一个包，就会自动取搜索这个包下的 init() 函数，并且执行。
- 如果这个 init() 函数做了一些非常重的事情，而且你的场景又用不到这些被 init 的东西，那么就会浪费资源，导致程序变慢。
- 如果 init() 函数做了一些针对生产环境的初始化，那么在你调试或者单测的时候就会出问题。

- 你可以以 Init 为前缀写一些特有的 Init 函数，例如上述例子中的 2，3 。这样不管是生产还是测试，都可以选择正确的 Init 函数。而且，也可以在你需要的时候在初始化(类似于懒汉模式)，这样也可以节省计算机资源。

#### 5. 合理使用注释

要写好的注释，而不是废话。

```go
// Bad example 1
func Abs(num int) int {
    // if num is negative
    if num < 0 {
        reutrn -num
    }
    // if num is non-negative
    return num
}

// Good example 1
// Return abosulote value of an int value
func Abs(num int) int {
    if num < 0 {
        reutrn -num
    }
    return num
}
```

- 上述例子，Bad example 中写得注释都是废话，没什么用，而且看代码就知道了。
- Good example 在函数前解释函数做了什么，让读者在阅读代码前就对该函数有了一个大体的认识。