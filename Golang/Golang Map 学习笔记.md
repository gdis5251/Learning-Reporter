# Golang Map 学习笔记

## Map 的数据结构

在源码中，map 的结构体叫 hmap，意思是 hashmap。

```go
// A header for a Go map.
type hmap struct {
    // 元素个数，调用 len(map) 时，直接返回此值
	count     int
	flags     uint8
	// buckets 的对数 log_2
	B         uint8
	// overflow 的 bucket 近似数
	noverflow uint16
	// 计算 key 的哈希的时候会传入哈希函数
	hash0     uint32
    // 指向 buckets 数组，大小为 2^B
    // 如果元素个数为0，就为 nil
	buckets    unsafe.Pointer
	// 扩容的时候，buckets 长度会是 oldbuckets 的两倍
	oldbuckets unsafe.Pointer
	// 指示扩容进度，小于此地址的 buckets 迁移完成
	nevacuate  uintptr
	extra *mapextra // optional fields
}
```

- B 是 buckets 数组长度的对数，意思是，2^B 就是 bucket 的数量。

buckets 是一个指针，指向一个结构体

```go
type bmap struct {
	tophash [bucketCnt]uint8
}
```

这只是表面，在编译期间会变成

```go
type bmap struct {
    topbits  [8]uint8
    keys     [8]keytype
    values   [8]valuetype
    pad      uintptr
    overflow uintptr
}
```

**map 数据结构整体图**

![image-20210705173047258](https://tva1.sinaimg.cn/large/008i3skNly1gs67duwhomj30m20fvtb5.jpg)

**再来看一下 bmap 的结构**

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

## 如何进行扩容

首先 golang 使用哈希表作为 map 的底层结构就是为了更快的找到 key。但是随着 key 的数量增加，key 发生碰撞的概率就越来越大。bucket 中 8 个 cell 会被逐渐塞满，那么查找、插入、删除 key 的效率就越来越低。

最理想的情况是一个 bucket 装一个 key，这样就能作为 O(1)的时间复杂度。但是这样太浪费空间，所以 golang 使用一个 bucket 装 8 个 key，这样又用时间换空间。但是也不能所有的 key 都落在同一个 bucket 里面，这样就退化成了链表，各种操作都会退化成 O(n) 的时间复杂度了。

因此，是否要进行扩容就需要有一个指标去判断。就有了「装载因子」这个指标。

```go
loadFactor := count / (2^B)
```

- count 就是 map 的元素个数
- 2^B 表示 bucket 的数量

map 在插入元素的时候，会先检测以下两个条件，如果满足条件才会触发扩容。

1. 装载因子超过阈值，源码里定义的阈值是 6.5。也就是说，平均每个 bucket 中元素的数量超过 6.5 就要扩容。
2. overflow 的 bucket 过多。其中有两种情况：当 B 小于 15，也就是 bucket 总数 2^B 小于 2^15 时，如果 overflow 的 bucket 数量超过 2^B；当 B >= 15，也就是 bucket 总数 2^B 大于等于 2^15，如果 overflow 的 bucket 数量超过 2^15。

赋值操作的对应代码：

```go
// 触发扩容时机
if !h.growing() && (overLoadFactor(int64(h.count), h.B) || tooManyOverflowBuckets(h.noverflow, h.B)) {
		hashGrow(t, h)
	}

// 装载因子超过 6.5
func overLoadFactor(count int64, B uint8) bool {
	return count >= bucketCnt && float32(count) >= loadFactor*float32((uint64(1)<<B))
}

// overflow buckets 太多
func tooManyOverflowBuckets(noverflow uint16, B uint8) bool {
	if B < 16 {
		return noverflow >= uint16(1)<<B
	}
	return noverflow >= 1<<15
}
```

解释一下：

第 1 点：每个 bucket 有 8 个空位，在没有溢出，且所有的桶都装满的情况下，装载因子算出来的结果是 8 。因此当装载因子超过 6.5 时，表示很多 bucket 快要装满了，查找效率和插入效率都变低了，这个时候扩容是正确的选择。

第 2 点：在装载因子较小的情况下，map 的查询效率也会比较低，但是第 1 点无法感知这种现象。可以理解为，装载因子比较小，即 map 的元素总数较少，但是 bucket 的数量比较多（bucket 包括普通的 bucket 和挂在 overflow 的 bucket）。所以当装载因子比较小的时候会触发重新分配的操作。

造成第 2 点的原因是：不停地插入、删除元素。先插入很多元素，导致创建了很多的 bucket，但是又没有达到装载因子的临界值，未触发扩容；后来又删除元素，降低了元素的总数量，再插入很多元素，导致创建了很多的 overflow bucket，还是不会触发第 1 点的规则，你能拿我怎么办？？？overflow bucket 的数量太多，导致 key 会很分散，查找插入的效率非常低，所以有了第 2 点的规定。

当命中 1、2 条件，都会发生扩容，准确的说第 2 点并不能称为扩容，应该叫重新分配key 的位置。

**扩容策略**

**对于第 1 点**： 元素太多，bucket 太少，很简单：将 B + 1，bucket 的数量(2 ^ B) 变为原来的两倍。于是就有了新老 bucket。注意：刚扩容完元素依然在老 bucket 里，还没有移到新 bucket 里，新 bucket 只是数量变为了老 bucket 的两倍。

**对于第 2 点**：元素不多，但是 bucket 特别多，说明 bucket 都没满。解决办法就是开辟一个大小与老 bucket 一样的空间，把老 bucket 的元素迁移到新的 bucket ，使其元素排列更加紧密。这样一来，原来在 overflow bucket 中的元素就可以启动到 bucket 中，节省空间，提高 bucket 的利用率，且 map 的查找和插入效率就会提升。

**迁移策略**：之所以不立刻迁移元素，如果元素过多，扩容完立刻迁移会造成性能的下降。那什么时候迁移呢？golang map 的新老 bucket 元素迁移不是一次性的，是「渐进式」的，只有在每次插入、修改、删除 key 的时候会尝试将老 bucket 中的元素迁移到新的 bucket 中，且每次最多迁移两个。这也就是为什么 map 结构体中有个表示迁移状态的字段。

**迁移具体逻辑**：

```go
func evacuate(t *maptype, h *hmap, oldbucket uintptr) {
	// 定位老的 bucket 地址
	b := (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
	// 结果是 2^B，如 B = 5，结果为32
	newbit := h.noldbuckets()
	// key 的哈希函数
	alg := t.key.alg
	// 如果 b 没有被搬迁过
	if !evacuated(b) {
		var (
			// 表示bucket 移动的目标地址
			x, y   *bmap
			// 指向 x,y 中的 key/val
			xi, yi int
			// 指向 x，y 中的 key
			xk, yk unsafe.Pointer
			// 指向 x，y 中的 value
			xv, yv unsafe.Pointer
		)
		// 默认是等 size 扩容，前后 bucket 序号不变
		// 使用 x 来进行搬迁
		x = (*bmap)(add(h.buckets, oldbucket*uintptr(t.bucketsize)))
		xi = 0
		xk = add(unsafe.Pointer(x), dataOffset)
		xv = add(xk, bucketCnt*uintptr(t.keysize))、

		// 如果不是等 size 扩容，前后 bucket 序号有变
		// 使用 y 来进行搬迁
		if !h.sameSizeGrow() {
			// y 代表的 bucket 序号增加了 2^B
			y = (*bmap)(add(h.buckets, (oldbucket+newbit)*uintptr(t.bucketsize)))
			yi = 0
			yk = add(unsafe.Pointer(y), dataOffset)
			yv = add(yk, bucketCnt*uintptr(t.keysize))
		}

		// 遍历所有的 bucket，包括 overflow buckets
		// b 是老的 bucket 地址
		for ; b != nil; b = b.overflow(t) {
			k := add(unsafe.Pointer(b), dataOffset)
			v := add(k, bucketCnt*uintptr(t.keysize))

			// 遍历 bucket 中的所有 cell
			for i := 0; i < bucketCnt; i, k, v = i+1, add(k, uintptr(t.keysize)), add(v, uintptr(t.valuesize)) {
				// 当前 cell 的 top hash 值
				top := b.tophash[i]
				// 如果 cell 为空，即没有 key
				if top == empty {
					// 那就标志它被"搬迁"过
					b.tophash[i] = evacuatedEmpty
					// 继续下个 cell
					continue
				}
				// 正常不会出现这种情况
				// 未被搬迁的 cell 只可能是 empty 或是
				// 正常的 top hash（大于 minTopHash）
				if top < minTopHash {
					throw("bad map state")
				}

				k2 := k
				// 如果 key 是指针，则解引用
				if t.indirectkey {
					k2 = *((*unsafe.Pointer)(k2))
				}

				// 默认使用 X，等量扩容
				useX := true
				// 如果不是等量扩容
				if !h.sameSizeGrow() {
					// 计算 hash 值，和 key 第一次写入时一样
					hash := alg.hash(k2, uintptr(h.hash0))

					// 如果有协程正在遍历 map
					if h.flags&iterator != 0 {
						// 如果出现 相同的 key 值，算出来的 hash 值不同
						if !t.reflexivekey && !alg.equal(k2, k2) {
							// 只有在 float 变量的 NaN() 情况下会出现
							if top&1 != 0 {
								// 第 B 位置 1
								hash |= newbit
							} else {
								// 第 B 位置 0
								hash &^= newbit
							}
							// 取高 8 位作为 top hash 值
							top = uint8(hash >> (sys.PtrSize*8 - 8))
							if top < minTopHash {
								top += minTopHash
							}
						}
					}

					// 取决于新哈希值的 oldB+1 位是 0 还是 1
					// 详细看后面的文章
					useX = hash&newbit == 0
				}

				// 如果 key 搬到 X 部分
				if useX {
					// 标志老的 cell 的 top hash 值，表示搬移到 X 部分
					b.tophash[i] = evacuatedX
					// 如果 xi 等于 8，说明要溢出了
					if xi == bucketCnt {
						// 新建一个 bucket
						newx := h.newoverflow(t, x)
						x = newx
						// xi 从 0 开始计数
						xi = 0
						// xk 表示 key 要移动到的位置
						xk = add(unsafe.Pointer(x), dataOffset)
						// xv 表示 value 要移动到的位置
						xv = add(xk, bucketCnt*uintptr(t.keysize))
					}
					// 设置 top hash 值
					x.tophash[xi] = top
					// key 是指针
					if t.indirectkey {
						// 将原 key（是指针）复制到新位置
						*(*unsafe.Pointer)(xk) = k2 // copy pointer
					} else {
						// 将原 key（是值）复制到新位置
						typedmemmove(t.key, xk, k) // copy value
					}
					// value 是指针，操作同 key
					if t.indirectvalue {
						*(*unsafe.Pointer)(xv) = *(*unsafe.Pointer)(v)
					} else {
						typedmemmove(t.elem, xv, v)
					}

					// 定位到下一个 cell
					xi++
					xk = add(xk, uintptr(t.keysize))
					xv = add(xv, uintptr(t.valuesize))
				} else { // key 搬到 Y 部分，操作同 X 部分
					// ……
					// 省略了这部分，操作和 X 部分相同
				}
			}
		}
		// 如果没有协程在使用老的 buckets，就把老 buckets 清除掉，帮助gc
		if h.flags&oldIterator == 0 {
			b = (*bmap)(add(h.oldbuckets, oldbucket*uintptr(t.bucketsize)))
			// 只清除bucket 的 key,value 部分，保留 top hash 部分，指示搬迁状态
			if t.bucket.kind&kindNoPointers == 0 {
				memclrHasPointers(add(unsafe.Pointer(b), dataOffset), uintptr(t.bucketsize)-dataOffset)
			} else {
				memclrNoHeapPointers(add(unsafe.Pointer(b), dataOffset), uintptr(t.bucketsize)-dataOffset)
			}
		}
	}

	// 更新搬迁进度
	// 如果此次搬迁的 bucket 等于当前进度
	if oldbucket == h.nevacuate {
		// 进度加 1
		h.nevacuate = oldbucket + 1
		// Experiments suggest that 1024 is overkill by at least an order of magnitude.
		// Put it in there as a safeguard anyway, to ensure O(1) behavior.
		// 尝试往后看 1024 个 bucket
		stop := h.nevacuate + 1024
		if stop > newbit {
			stop = newbit
		}
		// 寻找没有搬迁的 bucket
		for h.nevacuate != stop && bucketEvacuated(t, h, h.nevacuate) {
			h.nevacuate++
		}
		
		// 现在 h.nevacuate 之前的 bucket 都被搬迁完毕
		
		// 所有的 buckets 搬迁完毕
		if h.nevacuate == newbit {
			// 清除老的 buckets
			h.oldbuckets = nil
			// 清除老的 overflow bucket
			// 回忆一下：[0] 表示当前 overflow bucket
			// [1] 表示 old overflow bucket
			if h.extra != nil {
				h.extra.overflow[1] = nil
			}
			// 清除正在扩容的标志位
			h.flags &^= sameSizeGrow
		}
	}
}
```

总结一下：

先判断是哪种条件下的迁移：

如果是符合第 1 点的扩容，那么在决定 key 在哪个 bucket 里时，就取key 的 hash 值的低 B+1 位。

如果是符合第 2 点的扩容，与原来一样。

**map 遍历的无序特性**：

这样也就解释了 map 的遍历为什么是无序的了，因为如果发生了扩容，曾经在同一个 bucket 的元素可能就不在一起了。就像是你高中毕业了，那么你的大学同学很难有高中同学一样。

如果有人说，我就固定给 map 里写几个元素，那么遍历不就是一个伪有序的了吗？其实在 Go1.0 后版本，开发者在每次遍历时，都会从随机的位置开始遍历，并且在 bucket 里随机位置开始读元素，这么做的目的是为了防止新手程序员误以为 map 的遍历是有序的。我只能说：高！

## map 的遍历

map 的遍历看似很简单，就好像挨个遍历就行，但是复杂的地方是，如果遍历前发生过扩容，并且老 bucket 的元素还没有迁移完，这时对这个中间态的遍历是比较复杂的。

下面举一个遍历中间态的栗子🌰

假设有一个 map，刚开始 B = 1(也就是有两个 bucket)，后来发生了扩容(不纠结是否满足条件，只是为了举例子)，B 变成了 2，1 号 bucket 中的内容迁移到了新的 bucket，1 号 bucket 裂变成新 1 号和新 3 号。0 号 bucket 暂未迁移，老的 bucket 挂在 *oldbuckets 指针上，新的挂在 *buckets 上。

![image-20210701205215293](https://tva1.sinaimg.cn/large/008i3skNly1gs1qq91xxcj30kx09fjsz.jpg)

这时，我们对 map 进行遍历，假设初始化后，startBucket(初始 bucket 下标) = 3，offset(初始 cell 位置) = 2。那么就变成：

![image-20210701205349050](https://tva1.sinaimg.cn/large/008i3skNly1gs1qruevhxj30l609udhk.jpg)

标红表示起始位置，bucket 的遍历顺序为：3 -> 0 -> 1 -> 2

因为新 3 号对应的老 1 号(对应关系是 二进制的低 B - 1 位)，所以先检查老 1 号是否迁移完了，判断方法是：

```go
func evacuated(b *bmap) bool {
	h := b.tophash[0]
	return h > empty && h < minTopHash
}
```

如果 b.tophash[0] 的值在标志值范围内，即在 (0,4) 区间里，说明已经被搬迁过了。

```go
empty = 0
evacuatedEmpty = 1
evacuatedX = 2
evacuatedY = 3
minTopHash = 4
```

本例中老 1 号已经迁移完成，所以只需要遍历新 3 号的 bucket 即可。遍历完后得到：

![image-20210701205734385](https://tva1.sinaimg.cn/large/008i3skNly1gs1qvrbncyj308f046q2y.jpg)

因为新 3 号有 overflow bucket ，因此还需要遍历 overflow bucket ，得到结果：

![image-20210701205833174](https://tva1.sinaimg.cn/large/008i3skNly1gs1qwsebntj30ew049aae.jpg)

新 3 号遍历完后，到新 0 号，新 0 号对应老 0 号，先检查老 0 号是否迁移完成，发现没迁移完，遍历老 0 号。

注意：并不是把老 0 号的所有值取出来，因为老的 bucket 会裂变成两个新的 bucket，因此这里只会拿出原本要迁移到新 0 号的元素，也就是 key hash 值的低新 B 位，得到结果：

![image-20210701210144245](https://tva1.sinaimg.cn/large/008i3skNly1gs1r06bx8pj30k9042t9d.jpg)

然后遍历到新 1 号，和之前一样，发现老 1 号已经前已完成，遍历新 1 号得到：

![image-20210701210236113](https://tva1.sinaimg.cn/large/008i3skNly1gs1r0zcdcjj30l003x753.jpg)

遍历到新 2 号，和新 0 号一样的逻辑，得到结果：

![image-20210701210306054](https://tva1.sinaimg.cn/large/008i3skNly1gs1r1hp7kmj60kt03hwf302.jpg)

最后遍历到新 3 号时，发现所有的 bucket 都遍历完了，结束。

**map 的赋值和删除比较简单，因为篇幅原因，先不诺列了。**