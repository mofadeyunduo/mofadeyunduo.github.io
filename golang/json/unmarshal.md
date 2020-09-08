## 分析的 API

```
func Unmarshal(data []byte, v interface{}) error
```

## 做了什么事

1. 检查 Json 字符串是否合法
2. 通过反射设置字符串的内容

## 学到了什么

1. 设计模式：State 模式
2. 反射的用法

### State 模式

首先看下整个流程的 `scanner` 定义，`step` 为 `State` 的定义

```
type scanner struct {
	// The step is a func to be called to execute the next transition.
	// Also tried using an integer constant and a single func
	// with a switch, but using the func directly was 10% faster
	// on a 64-bit Mac Mini, and it's nicer to read.
    // step 的类型其实就是 State 的定义，利用函数保存了算法
	step func(*scanner, byte) int

	// Reached end of top-level value.
	endTop bool

	// Stack of what we're in the middle of - array values, object keys, object values.
	parseState []int

	// Error that happened, if any.
	err error

	// total bytes consumed, updated by decoder.Decode (and deliberately
	// not set to zero by scan.reset)
	bytes int64
}
```

接着看下 step 是如何调用的：

```
// scanWhile processes bytes in d.data[d.off:] until it
// receives a scan code not equal to op.
func (d *decodeState) scanWhile(op int) {
	s, data, i := &d.scan, d.data, d.off
	for i < len(data) {
        // 每一次会调用 step 方法解析 JSON 字符串，如果解析到 JSON 定义格式的边界（比如遇到了 : 号的状态）
        // 会把 s.step 重新赋值，变为下个格式的解析（遇到 ：号之后，进入解析值的状态）
		newOp := s.step(s, data[i])
		i++
		if newOp != op {
			d.opcode = newOp
			d.off = i
			return
		}
	}

	d.off = len(data) + 1 // mark processed EOF with len+1
	d.opcode = d.scan.eof()
}
```

再看下一些 `State` 的算法：

```
// stateBeginString is the state after reading `{"key": value,`.
// 匹配到 {"key": value, 时的算法
func stateBeginString(s *scanner, c byte) int {
    // 下一个字符串是空格，下一个 State 是 scanSkipSpace
	if c <= ' ' && isSpace(c) {
		return scanSkipSpace
	}
    // 下一个字符串是 "，说明是文字，就进入 scanBeginLiteral 处理
	if c == '"' {
		s.step = stateInString
		return scanBeginLiteral
	}
    // 下一个字符不合法，状态错误
	return s.error(c, "looking for beginning of object key string")
}
```

总的来说，可以这么理解流程，比如说假设解析无限递归的 `{"a":{"b":{"c":...}}}`：

```         
stateBeginValue 开始解析 <-----------------------------------------|
         |                                                        |
stateBeginStringOrEmpty 为 { 开始解析                              |
         |                                                        |
stateInString 为 " 解析字符串                                       |
         |                                                        |
stateEndValue 为 " 值解析完成，                                     |
         |                                                        |
stateBeginValue : 之后，开始解析 Value, -----------------------------
```

也有一些具体的[状态图](https://www.json.org/json-zh.html)可以参考。

### 反射的用法

至于反射倒是没什么可说的，细说遇到的一些理解，有的可能比较奇怪：

1. Golang 的对象是分为 `Value` 和 `Type` 的，`Value` 就是实际存储的值，`Type` 是它的类型；
当传入 `interface{}` 类型时，保存了 `Type`，就知道了 obj 的具体类型
1. `ValueOf(i)` 返回的是一个 obj 的 copy，但这个 copy 是无法修改的，必须调用 `Elem()` 才可以修改；
但是 `Elem()` 只有在是 `ptr` 的时候才可以调用，所以如果要修改 obj 的值，必须传入 `ptr`
这么做是因为，Golang 函数是值拷贝，这个时候如果让用 `ValueOf(i)` 得到的值可以修改 obj，
实际上在函数调用者那里并没有修改 obj 的值，会让人很困惑，所以 ValueOf 如果不是个 `ptr`，就不让修改，避免这种迷惑行为
1. 通过 `CanSet()` 方法可以获取该值是否可以修改
1. `New(typ Type)` 获取的值类型 typ 的指针，多了一层指针
1. `reflect.Type.Elem()` 和 `reflect.Value.Elem()` 的含义不一致，`Type` 返回的是类型的具体值，比如 `[]int` 会返回 `int`；
`Value` 会返回 `ptr` 指向的内容

## 可以优化的地方

1. 为什么要先验证合法性才解析呢？看了文档的说明是不想部分解析。其实如果允许部分解析的话性能应该会提升
