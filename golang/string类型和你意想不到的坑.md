# 1. `Go`的string 类型

字符串是我们在编程中使用频繁的类型。在`Go`中，字符串一般被当作一个不可修改的整体。它的底层由一个`StringHeader`结构体和一个字符数组来实现。其中`StringHeader`的定义：

```go
type StringHeader struct {
	Data uintptr
	Len  int
}
```

`Data`是一个指向字符数组的指针，Len表示**字符数组的长度**。

> 请特别注意上面标成粗体的部分：**字符数组的长度**，我们后文会再次提到。

程序一般把字符串分配到只读内存空间，这块内存中的内容不会被修改。但是在运行时，我们还是可以获取它的拷贝，放到堆或者栈上，把字符串转换成字符数组，就可以修改其中的值。

从平时的使用来看，字符串类型简单好用，“人畜无害”，似乎是比较安全的变量类型。但是从其底层实现来看，字符串上的操作还是隐藏了不少的坑，有些坑甚至对程序的性能有显著的影响。

比较容易对程序造成影响的字符串操作有以下三方面：

1. 字符串的判等(比较)
2. 字符串的拼接操作
3. 字符串与字符数组的互相转换

# 2. 字符串判等

先从一段代码看起

```go
func main() {
	a := make([]byte, 16)
	a[0] = 'a'

	aStr := string(a)
    
	fmt.Println(aStr == "a")
	fmt.Println(len(aStr))
}
```

在你还没有运行这段代码前，请先预测一下可能的输出结果。

有不少人可能会认为输出结果应该是：

```
true
1
```

但实际并不是。在上面程序的第5行`aStr := string(a)`这里发生了类型转换，将一个字符数组转换成了字符串。那么这个转换是值拷贝？还是引用传递？我们需要看一下转型过程中实际调用的函数，字符数组转换成字符串时调用了`slicebytetostringtmp`（代码在`src/runtime/string.go`）。

来看看`slicebytetostringtmp`的实现

```go
func slicebytetostringtmp(b []byte) string {
	if raceenabled && len(b) > 0 {
		racereadrangepc(unsafe.Pointer(&b[0]),
			uintptr(len(b)),
			getcallerpc(),
			funcPC(slicebytetostringtmp))
	}
	if msanenabled && len(b) > 0 {
		msanread(unsafe.Pointer(&b[0]), uintptr(len(b)))
	}
	return *(*string)(unsafe.Pointer(&b))
}
```

可以看到，在调用`slicebytetostringtmp`时先发生值拷贝，把一个字符数组的值拷贝到变量`b []byte`  中。然后取`b`的地址作为`StringHeader`中`Data`的值，取`len(b)`作为字符串的长度。

其实看到这里大家也就明白了: 在执行完`aStr := string(a)`以后，`aStr`的`StringHeader`中`Len`并不是1，而是16. 这里比较反直觉，如果我的字符数组后面的值无效，为什么要记录到`Len`中？ 我们 直观上认为`aStr`的合理长度应当为1. 但是不清楚这是官方的不严谨，还是有意为之，这里的`Len`确实没有反应`aStr`的实际有效长度。正如本文开头我们特别强调的：**`StringHeader 中的Len是字符数组的长度`，而不是字符串的有效长度**

当判断两个字符串是否相等时，实际比较的是底层的字符数组。在上面例子中`"a"`这个字符串长度确实是1，而且它的底层，字符数组为`['a']`

而`aStr`字符串的字符数组实际是`['a', '0', '0', ..., '0']`。在比较`aStr == "a"`是，因为底层字符数组长度不一样，所以两者是不相等的。

# 3. 字符串拼接

字符串拼接有五种常见的拼接方法:

1. 使用"+"对字符串拼接
2. 使用`fmt.Sprintf()`对字符拼接
3. 使用`Join`拼接
4. 使用`bytes.Buffer`拼接
5. 使用`strings.Builder`拼接

无论哪种拼接方法，底层都会调用到`concatstrings()`函数

```go
func concatstrings(buf *tmpBuf, a []string) string {
	idx := 0
	l := 0
	count := 0
	for i, x := range a {
		n := len(x)
		if n == 0 {
			continue
		}
		l += n
		count++
		idx = i
	}
	if count == 0 {
		return ""
	}
	if count == 1 && (buf != nil || !stringDataOnStack(a[idx])) {
		return a[idx]
	}
	s, b := rawstringtmp(buf, l)
	for _, x := range a {
		copy(b, x)
		b = b[len(x):]
	}
	return s
}
```

从上面的代码可以看到，基本上拼接后的字符串完全是由其他字符串拷贝过来的。如果需要拼接的字符串非常多或非常大，拷贝造成的性能损失就比较严重。

# 3. 类型转换

在第二小节中，我们讨论过字符数组转型成字符串的过程，特别注意到：字符串中Len的值有可能与实际有效长度不一致。

如果要把一个字符串转换成字符数组呢？下面是字符串转成字符数组的代码

```go
func stringtoslicebyte(buf *tmpBuf, s string) []byte {
	var b []byte
	if buf != nil && len(s) <= len(buf) {
		*buf = tmpBuf{}
		b = buf[:len(s)]
	} else {
		b = rawbyteslice(len(s))
	}
	copy(b, s)
	return b
}
```

可以看到，`stringtoslicebyte()`函数内部先声明一个`b []byte`数组，然后判断字符串长度和给定的 buf的长度，然后把字符串中的数组逐个拷贝到`b`数组中。

# 4. 总结

1. 在编程中应当注意字符串拼接和转型操作，因为这些操作的底层可能会有大量的拷贝。
2. 相比于`string`类型，`[]byte`的行为更加受控和符合预期。 而`string`类型的长度有可能并没有反应它的真实有效长度。
3. 在不需要对数据进行修改的地方，可以优先使用`string`。如果你的数据需要修改，尽量在声明时就指定为字符数组，而不是在你需要修改的地方临时把字符串转型成字符数组。
4. 代码中应尽量减少字符串与字符数组的转型

# 参考

https://draveness.me/golang/docs/part2-foundation/ch03-datastructure/golang-string/#344-%E7%B1%BB%E5%9E%8B%E8%BD%AC%E6%8D%A2

https://www.flysnow.org/2018/10/28/golang-concat-strings-performance-analysis.html