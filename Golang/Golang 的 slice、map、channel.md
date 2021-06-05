## slice vs array

### 1. array

数据是 Golang 里最基本的类型，它的类型由元素类型和元素个数共同决定，因此 [3]int 和 [4]int 是两种类型。数据的类型在编译期间就决定了，**数据的元素个数是常量**，因此数组是无法扩容的。

Golang 有两种数据声明方式

```go
// 1. 显式指定数据的长度
array1 := [3]int{1, 2, 3}
// 2. 由初始化的元素个数来推导数据长度
array2 := [...]int{1, 2, 3}
```

array2 的数组元素个数由初始化的元素个数决定，且在编译期间会把第二种形式自动转为第一种形式。

---

### 2. slice

切片可以理解成是动态数组，因此它的长度是不固定的。**切片的长度是一个变量存放在栈上，**我们一般通过 append 方式来向切片中添加元素，当切片的容量不足时会自动扩容。

**slice 底层结构**

查看 reflect 包中切片的构成如下：

```go
// SliceHeader is the runtime representation of a slice.
// It cannot be used safely or portably and its representation may
// change in a later release.
// Moreover, the Data field is not sufficient to guarantee the data
// it references will not be garbage collected, so programs must keep
// a separate, correctly typed pointer to the underlying data.
type SliceHeader struct {
	Data uintptr
	Len  int
	Cap  int
}
```

可以看出，切片本质上是对数组进行了一次封装，多个切片之间可以共享底层的数据。切片操作超出cap(slice)的上限将导致一个panic异常，但是超出len(slice)则是意味着扩展了slice，因为新slice的长度会变大：

```go
array := [...]int{0, 1, 2, 3, 4, 5, 6, 7}
slice := array[3:5]
fmt.Println(slice[:7]) // panic: out of range
newSlice := slice[:5] // extend a slice (within capacity)
fmt.Println(newSlice) // "[3 4 5 6 7]"
```

当切片容量不足时，会调用 growslice 来对切片进行扩容，扩容会分配新的内存，并且深拷贝切片的所有元素。

```go
func growslice(et *_type, old slice, cap int) slice {
  newcap := old.cap
  doublecap := newcap + newcap
  if cap > doublecap {
    newcap = cap
  } else {
    if old.len < 1024 {
      newcap = doublecap
    } else {
      for 0 < newcap && newcap < cap {
        newcap += newcap / 4
      }
      if newcap <= 0 {
        newcap = cap
      }
    }
}
```

- 如果期望容量大于当前容量的两倍就会使用期望容量；

- 如果当前切片的长度小于 1024 就会将容量翻倍；

- 如果当前切片的长度大于 1024 就会每次增加 25% 的容量，直到新容量大于期望容量；

上述代码片段仅会确定切片的大致容量，下面还需要根据切片中的元素大小对齐内存，当数组中元素所占的字节大小为 1、8 或者 2 的倍数时会向上取整内存的大小，从而提高内存的分配效率并减少碎片，向上取整规则如下：

```go
const _NumSizeClasses = 67

var class_to_size = [_NumSizeClasses]uint16{0, 8, 16, 32, 48, 64, 80, 96, 112, 128, 144, 160, 176, 192, 208, 224, 240, 256, 288, 320, 352, 384, 416, 448, 480, 512, 576, 640, 704, 768, 896, 1024, 1152, 1280, 1408, 1536,1792, 2048, 2304, 2688, 3072, 3200, 3456, 4096, 4864, 5376, 6144, 6528, 6784, 6912, 8192, 9472, 9728, 10240, 10880, 12288, 13568, 14336, 16384, 18432, 19072, 20480, 21760, 24576, 27264, 28672, 32768}
```

#### tips-slice 的小坑

```go
var s []int        // len(s) == 0, s == nil
s = []int(nil)     // len(s) == 0, s == nil
s = []int{}        // len(s) == 0, s != nil
s = make([]int, 0) // len(s) == 0, s != nil
```

以上有 4 种 slice 声明或初始化的方式，但是这 4 中方式在 json.Marshal 的分为两类。

前两种会打印 ```"nil"```

后两种会打印```"[]"```

所以同学们在使用时一定要看仔细自己 slice 的声明方式，否则会带来预期之外的结果。

---

#### array vs slice

- array 需要指明长度，长度为常量且不可改变
- array 长度为其类型中的组成部分
- array 在作为函数参数的时候会产生 copy
- golang 所有函数参数都是值传递

---

**slice 的 len 和 cap 的增长策略**

- 当原 cap 小于 1024，每次cap *= 2
- 当原 cap 大于 1024，每次 cap * 1.25

该策略经过验证效率最优

---

**什么情况下最好要使用array**

- array是可hash的，而slice不是，因为slice的底层ArrayPtr是可变的，【这似乎是为什么一定要使用array而不是slice的唯一场景，参考资料详见：https://www.reddit.com/r/golang/comments/ecqgha/why_are_slices_in_go_unhashable/，https://stackoverflow.com/questions/30694652/why-use-arrays-instead-of-slices】

---

**优化点**

- slice 预先分配内存可以提升性能
- slice 直接使用 index 赋值而非 append 可以提升性能

---

**bce 优化**

```go
s1 := make([]int, 100)
.....


// bce 优化
_ = s1[len(s1) - 1]
for index, num := s1 {
  // do something
}
```

Golang 编译器在遍历 slice 时，在汇编层面，会先判断一下该下标是否存在，如果存在再进行取值。而 bce 优化就是先告诉编译器该 slice 最长是多少，然后再遍历时，编译器就不需要去判断该下标是否存在了，从而提升性能。

