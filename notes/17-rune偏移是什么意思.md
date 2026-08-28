# 17 · rune 偏移是什么意思(字节 vs rune vs 行号)

> 用户问:rune 偏移是什么意思?

---

## 一句话

> **rune = 一个 Unicode 字符**(不管这字符在内存里占几个字节)
> **rune 偏移 = 第几个字符**(从 0 开始数,一个汉字算 1,一个字母也算 1)

---

## 对比 3 种"偏移"

假设字符串是 `"你好A"`(2 个汉字 + 1 个字母):

### 1. 字节偏移(byte offset)—— 内存里的第几个字节

Go 的字符串是 UTF-8 编码:
```
"你好A"
 ↓
字节: [E4 BD A0] [E5 A5 BD] [41]
位置:  0 1 2     3 4 5     6

你 = 字节 0~2(3 个字节)
好 = 字节 3~5(3 个字节)
A  = 字节 6(1 个字节)
```

**字节偏移**:你=0,好=3,A=6。**总长度 7 字节**。

### 2. rune 偏移(rune offset)—— 第几个字符

```
"你好A"
 ↓
字符: 你  好  A
位置:  0   1   2

你 = 第 0 个字符
好 = 第 1 个字符
A  = 第 2 个字符
```

**rune 偏移**:你=0,好=1,A=2。**总长度 3 个 rune**。

### 3. 行号(line number)—— 第几行

跟字节/rune 都不同,按换行符 `\n` 切分后是第几行。这个项目**保护区间不用行号**(因为同行的不同位置偏移无法区分)。

---

## 为什么要用 rune 偏移,不用字节偏移

### 问题:字节偏移对中文有歧义

```
chunk.Start = 3  ← 这是字节 3,是"好"的开头,还是"你"的中间?
```

中文 1 个字 = 3 字节,字节偏移 3 落在"你"的中间(字节 1、2 是"你"的后半部分),切这里会把"你"切坏。

### 解法:rune 偏移直接对应字符

```
chunk.Start = 1  ← 第 1 个字符,就是"好"
```

rune 偏移**永远是字符边界**,不会切在字符中间。

---

## 代码里怎么数 rune

```go
func runeLen(s string) int {
    return utf8.RuneCountInString(s)  // 数 rune 数量
}

// 例:runeLen("你好A") = 3
```

```go
// 字符串转 rune 数组
runes := []rune("你好A")  // [你, 好, A]
runes[0]  // 你
runes[1]  // 好
runes[2]  // A
```

---

## 这个项目里 rune 偏移用在哪

### 1. Chunk 的 Start/End 都是 rune 偏移

```go
type Chunk struct {
    Content string
    Start   int  // rune 偏移
    End     int  // rune 偏移
}
```

**位置不变量**:
```go
End - Start == utf8.RuneCountInString(Content)
// 例:chunk "你好A" 的 Start=10,End=13,Content 长度 = 3 个 rune
// End - Start = 3 = runeCount("你好A") ✓
```

### 2. 切分逻辑用 rune 偏移

```go
runes := []rune(text)
chunk.Content = string(runes[start:end])  // 切 [start, end) 这段 rune
```

**为什么**:切 [10, 13) 直接对应"第 10、11、12 个字符",不会切坏中文字符。

### 3. 保护区间用字节偏移,但转换到 rune

```go
// 正则返回字节偏移(快)
byteSpans := protectedSpans(text)  // span{start: 0, end: 50}(字节)

// 转成 rune 偏移(给切分逻辑用)
runeSpans := protectedSpansRune(text, byteSpans)  // span{start: 0, end: 30}(rune)
```

**为什么这么设计**:
- 正则匹配按字节工作(快,Go 的 regexp 返回字节偏移)
- 切分逻辑按字符工作(正确,中文 1 字 = 1 rune)
- 转换一次性

---

## 一个具体例子

**文档**:`"前言\n\n$$E=mc^2$$\n\n后语"`

### 字节视角(简化)

```
前 言 \n \n $ $ E = m c ^ 2 $ $ \n \n 后 语
0  3  6  7  8  9 10 11 12 13 14 15 16 17 18 19 20 21 22 25
```

(注:"前"占字节 0-2,"言"占 3-5,`\n` 是 1 字节)

### rune 视角

```
前 言 \n \n $ $ E = m c ^ 2 $ $ \n \n 后 语
0  1  2  3  4  5  6  7  8  9 10 11 12 13 14 15 16 17
```

### 保护区间

正则找 `$$...$$` 返回**字节偏移**:[9, 17)

转换成 **rune 偏移**:[4, 14)

### 切分时

切分逻辑用 rune 偏移:
```go
runes := []rune(text)
// 保护区间 [4, 14) 是原子,不能切
// 切 [0, 4) = "前言\n\n" (rune 0-3)
// 切 [4, 14) = "$$E=mc^2$$\n\n" (rune 4-13,原子)
// 切 [14, 末尾) = "后语"
```

---

## 你应该带走的 3 件事

1. **rune = 一个 Unicode 字符**(汉字、字母、符号都算 1 个 rune),不管内存占几字节。**rune 偏移 = 第几个字符**(从 0 开始)。

2. **字节偏移 vs rune 偏移**:中文 1 字 = 3 字节,所以字节偏移和 rune 偏移**数值不一样**。字节偏移可能切在字符中间,rune 偏移永远是字符边界,**切分用 rune 偏移才能不切坏中文**。

3. **这个项目的设计**:正则匹配用字节偏移(快,Go regexp 返回字节),切分逻辑用 rune 偏移(正确,中文不切坏),`protectedSpansRune` 做转换。Chunk 的 Start/End 都是 rune 偏移,保证 `End - Start = runeCount(Content)` 不变量。