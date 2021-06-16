# Golang Map 学习笔记

![image-20210616155012269](https://tva1.sinaimg.cn/large/008i3skNly1grk6zr5wlcj30la0p9ju0.jpg)

- 第一行粉色的是一个有 8 个元素的哈希数组，只会起到占位的作用，表示第几个位置已经有值了。
- Key 和 value 在对应有值的位置填补值。
- Key 和 value 是分开放的，这样可以省掉padding，节省内存空间。

> Eg: 比如现在有一个 map[int64]int8 ，如果按照 key-value/key-value 的形式存放，那么每一组 kv 之间都需要补齐 7 个字节，会大量浪费内存空间。

> Tips: int64 占 8 个字节，int8 占 1 个字节，由于内存对齐机制，他们放在一起内存会自动对齐到最大内存元素的整数倍也就是 16 个字节，所以需要补 7 个字节。

- 每个 bucket 只能存放 8 个 k-v 对，如果需要存放更多，那么就需要再构建新的 bucket 通过 overflow 连接起来。

## 创建 map

语法层面：

```go
// 使用 make 正确创建 map
m1 := make(map[string]int)

m2 := make(map[string]int, 7)

// 错误方式
var m3 map[string]int

m4 := new(map[string]int)

```

错误的创建 map 方式是创建一块内存大小为 map[string]int 的内存，然后值类型为零值，指针类型为空指针，所以，在错误的创建方式下，如果直接操作 map 会 panic，报空指针异常。

而使用 make 的方式创建 map，底层实际调用了 makemap 函数，主要工作是初始化 hmap 结构体的各种字段。

```go
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap {
	// 省略各种条件检查...

	// 找到一个 B，使得 map 的装载因子在正常范围内
	B := uint8(0)
	for ; overLoadFactor(hint, B); B++ {
	}

	// 初始化 hash table
	// 如果 B 等于 0，那么 buckets 就会在赋值的时候再分配
	// 如果长度比较大，分配内存会花费长一点
	buckets := bucket
	var extra *mapextra
	if B != 0 {
		var nextOverflow *bmap
		buckets, nextOverflow = makeBucketArray(t, B)
		if nextOverflow != nil {
			extra = new(mapextra)
			extra.nextOverflow = nextOverflow
		}
	}

	// 初始化 hamp
	if h == nil {
		h = (*hmap)(newobject(t.hmap))
	}
	h.count = 0
	h.B = B
	h.extra = extra
	h.flags = 0
	h.hash0 = fastrand()
	h.buckets = buckets
	h.oldbuckets = nil
	h.nevacuate = 0
	h.noverflow = 0

	return h
}
```

makeMap 返回的是指针，所以当 map 作为函数参数时，在函数内部操作 map 会影响 map 自身。

## key 定位过程

key 经过哈希计算后得到哈希值，共 64 个 bit 位（不讨论 32 位机）

- 定位桶：定位桶只会用到后 B 个位，例如 B 是 5，那么桶的数量也是就 buckets 数组的长度是 2^5 = 32 个

  例如现在有一个 key 经过哈希计算后得到

  ```shell
  10010111 | 000011110110110010001111001010100010010110010101010 │ 01010
  ```

  后 5 位是 10，那么就定位到 10 号桶。

- 定位 key：使用哈希值的高 8 位来定位 key 在 bucket 中的位置，这里是寻找已有的 key。最开始桶内没有 key，新加入的 key 会找到第一个空位，放入。

  buckets 编号就是桶编号，当两个不同的 key 落在同一个桶中，也就是发生了哈希冲突。冲突的解决手段是用链表法：在 bucket 中，从前往后找到第一个空位。这样，在查找某个 key 时，先找到对应的桶，再去遍历 bucket 中的 key。

  ![image-20210616163437945](https://tva1.sinaimg.cn/large/008i3skNly1grk6zl10fpj30nj0q40vd.jpg)

以上图为例，B 为 5，取后 5 位来定位到 6 号桶，然后再取前 8 位，得到 151。遍历 tophash，找到值为 151 的位置。

如果在 bucket 中没找到，并且 overflow 不为空，还要继续去 overflow 中去找，直到找到或者所有的 key 槽位都找遍了包括 overflow。

以下是 mapaccess1函数源码

```go
func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer {
	// ……
	
	// 如果 h 什么都没有，返回零值
	if h == nil || h.count == 0 {
		return unsafe.Pointer(&zeroVal[0])
	}
	
	// 写和读冲突
	if h.flags&hashWriting != 0 {
		throw("concurrent map read and map write")
	}
	
	// 不同类型 key 使用的 hash 算法在编译期确定
	alg := t.key.alg
	
	// 计算哈希值，并且加入 hash0 引入随机性
	hash := alg.hash(key, uintptr(h.hash0))
	
	// 比如 B=5，那 m 就是31，二进制是全 1
	// 求 bucket num 时，将 hash 与 m 相与，
	// 达到 bucket num 由 hash 的低 8 位决定的效果
	m := uintptr(1)<<h.B - 1
	
	// b 就是 bucket 的地址
	b := (*bmap)(add(h.buckets, (hash&m)*uintptr(t.bucketsize)))
	
	// oldbuckets 不为 nil，说明发生了扩容
	if c := h.oldbuckets; c != nil {
	    // 如果不是同 size 扩容（看后面扩容的内容）
	    // 对应条件 1 的解决方案
		if !h.sameSizeGrow() {
			// 新 bucket 数量是老的 2 倍
			m >>= 1
		}
		
		// 求出 key 在老的 map 中的 bucket 位置
		oldb := (*bmap)(add(c, (hash&m)*uintptr(t.bucketsize)))
		
		// 如果 oldb 没有搬迁到新的 bucket
		// 那就在老的 bucket 中寻找
		if !evacuated(oldb) {
			b = oldb
		}
	}
	
	// 计算出高 8 位的 hash
	// 相当于右移 56 位，只取高8位
	top := uint8(hash >> (sys.PtrSize*8 - 8))
	
	// 增加一个 minTopHash
	if top < minTopHash {
		top += minTopHash
	}
	for {
	    // 遍历 8 个 bucket
		for i := uintptr(0); i < bucketCnt; i++ {
		    // tophash 不匹配，继续
			if b.tophash[i] != top {
				continue
			}
			// tophash 匹配，定位到 key 的位置
			k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))
			// key 是指针
			if t.indirectkey {
			    // 解引用
				k = *((*unsafe.Pointer)(k))
			}
			// 如果 key 相等
			if alg.equal(key, k) {
			    // 定位到 value 的位置
				v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
				// value 解引用
				if t.indirectvalue {
					v = *((*unsafe.Pointer)(v))
				}
				return v
			}
		}
		
		// bucket 找完（还没找到），继续到 overflow bucket 里找
		b = b.overflow(t)
		// overflow bucket 也找完了，说明没有目标 key
		// 返回零值
		if b == nil {
			return unsafe.Pointer(&zeroVal[0])
		}
	}
}
```

这个代码不太好看，但是结合上面的逻辑再多看几遍就懂了。

**部分源码解读**

**key value 定位公式**

```go
// key 定位公式
k := add(unsafe.Pointer(b), dataOffset+i*uintptr(t.keysize))

// value 定位公式
v := add(unsafe.Pointer(b), dataOffset+bucketCnt*uintptr(t.keysize)+i*uintptr(t.valuesize))
```

b 是 bmap 的地址，这里 bmap 还是源码里定义的结构体，只包含一个 tophash 数组，经编译器扩充之后的结构体才包含 key，value，overflow 这些字段。dataOffset 是 key 相对于 bmap 起始地址的偏移：

```go
dataOffset = unsafe.Offsetof(struct {
		b bmap
		v int64
	}{}.v)
```

因此 bucket 里 key 的起始地址就是 unsafe.Pointer(b)+dataOffset。第 i 个 key 的地址就要在此基础上跨过 i 个 key 的大小；而我们又知道，value 的地址是在所有 key 之后，因此第 i 个 value 的地址还需要加上所有 key 的偏移。理解了这些，上面 key 和 value 的定位公式就很好理解了。

**大循环**

```go
b = b.overflow(t)
```

遍历所有的 bucket，这相当于是一个 bucket 链表。

当定位到一个具体的 bucket 时，里层循环就是遍历这个 bucket 里所有的 cell，或者说所有的槽位，也就是 bucketCnt=8 个槽位。整个循环过程：

![image-20210616165451897](https://tva1.sinaimg.cn/large/008i3skNly1grk7kkxykfj30lt0am75l.jpg)

**minTopHash**

当一个 cell 的 tophash 值小于 minTopHash 时，标志这个 cell 的迁移状态。因为这个状态值是放在 tophash 数组里，为了和正常的哈希值区分开，会给 key 计算出来的哈希值一个增量：minTopHash。这样就能区分正常的 top hash 值和表示状态的哈希值。

下面的这几种状态就表征了 bucket 的情况：

```go
// 空的 cell，也是初始时 bucket 的状态
empty          = 0
// 空的 cell，表示 cell 已经被迁移到新的 bucket
evacuatedEmpty = 1
// key,value 已经搬迁完毕，但是 key 都在新 bucket 前半部分，
// 后面扩容部分会再讲到。
evacuatedX     = 2
// 同上，key 在后半部分
evacuatedY     = 3
// tophash 的最小正常值
minTopHash     = 4
```

源码里判断这个 bucket 是否已经搬迁完毕，用到的函数：

```go
func evacuated(b *bmap) bool {
	h := b.tophash[0]
	return h > empty && h < minTopHash
}
```

只取了 tophash 数组的第一个值，判断它是否在 0-4 之间。对比上面的常量，当 top hash 是 `evacuatedEmpty`、`evacuatedX`、`evacuatedY` 这三个值之一，说明此 bucket 中的 key 全部被搬迁到了新 bucket。

## map 的两种 get

map 有两种 get 方式，带 comma 和不带 comma，我本来觉得这个太神奇了吧，golang 的函数如果有两个返回值就算不接收也要用下划线代替，但是这里就可接收可不接收。原来这是编译器做的工作，编译器在分析代码后，将 map 的 get 对应到了两个不同的函数上。

```go
// src/runtime/hashmap.go

func mapaccess1(t *maptype, h *hmap, key unsafe.Pointer) unsafe.Pointer

func mapaccess2(t *maptype, h *hmap, key unsafe.Pointer) (unsafe.Pointer, bool)
```

源码真的是不拘小节，直接用 1和 2 后缀来区分，两个函数的实现基本一样，只是返回值不一样。

另外，根据 key 的不同类型，编译器还会将查找、插入、删除的函数用更具体的函数替换，以优化效率：

| key 类型 | 查找                                                         |
| -------- | ------------------------------------------------------------ |
| uint32   | mapaccess1_fast32(t *maptype, h *hmap, key uint32) unsafe.Pointer |
| uint32   | mapaccess2_fast32(t *maptype, h *hmap, key uint32) (unsafe.Pointer, bool) |
| uint64   | mapaccess1_fast64(t *maptype, h *hmap, key uint64) unsafe.Pointer |
| uint64   | mapaccess2_fast64(t *maptype, h *hmap, key uint64) (unsafe.Pointer, bool) |
| string   | mapaccess1_faststr(t *maptype, h *hmap, ky string) unsafe.Pointer |
| string   | mapaccess2_faststr(t *maptype, h *hmap, ky string) (unsafe.Pointer, bool) |

这些函数的参数类型直接是具体的 uint32、unt64、string，在函数内部由于提前知晓了 key 的类型，所以内存布局是很清楚的，因此能节省很多操作，提高效率。

上面这些函数都是在文件 `src/runtime/hashmap_fast.go` 里。

## 占坑

- 同时读写 map