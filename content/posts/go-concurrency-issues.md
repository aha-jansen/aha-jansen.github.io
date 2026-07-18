---
title: "流式输出场景下的字符串替换——一次颇具挑战性的需求"
date: 2023-09-02
draft: false
tags:
  - 技术
  - Go
categories:
  - 技术
---

## 背景

最近在赶项目的过程中出现了一个需求，有多段带有引用的文本，为了防止引用之间出现索引冲突，需要登记并重新编排所有引用。随后会收到一个 `chan string`， 需要将输入的文本中的索引用替换成超链接的格式。大概抽象出来的问题可以表述成如下：

要求实现一个引用管理的模块 `refManager` ： 

对外提供一个 `Recv(in chan)` 接口，将得到的文本按照 `refs` 中的 `(k,v)` 将 `k` 替换成 `k+v`，例如

`refs` 中包含 `{"ab": "1", "e", "2"}` ，那么：

```
case1: abcdefg -> ab1cde2fg
case2: aabb -> aab1b
```

可以保证的是，任意 `v` 不包含任意  `k` ，因此不需要担心循环替换的情况出现，正式的表达为： $k_i \notin v_j | 0 < i, j < len(refs)$ 。

```golang
type refManager struct{
    refs map[string]string
}

func (m *refManager) Recv(in <-chan string, out chan string) {
}
```

`in, out` 分别表示生产者管道和消费者管道，都是字符串类型。

这里一开始有两个思路，后面分析这两个思路的时间复杂度和空间复杂度：

假设 `Recv()` 中的输入管道 `in` 发送 `n` 个字符串 `s` ，每个字符串平均长度为 `m`，其中：

$0 < len(k) < 4; 0 < m < 100; 0<len(refs)<100; 0<n<1000$

## 方法一：维护字符串

给定一个字符串列表 `buf` ，假设 $L=max(len(k_i), k_i \in refs)$，`buf` 固定维护`L` 个 `s`。方法一的思想：当收到的字符串个数大于 `L` 时，去查找是否存在子字符串等于 $k_i$ ，然后将 `buf` 合并成一个字符串 `allS` ，然后替换所有的`(k,v)`。代码如下：

```go

type refManager struct {
	refs map[string]string
}

func (m *refManager) replaceAll(s string) string {
	newS := s
	for k, v := range m.refs {
		newS = strings.ReplaceAll(newS, k, v)
	}
	return newS
}

func (m *refManager) Recv(in <-chan string, out chan string) {
	L := 4
	tail := 0
	buf := make([]string, L)
	for s := range in {
		// buf overflow, remove the first, and the first element cannot be the part of k
		if tail >= L {
			out <- buf[0]
			for i := 1; i < tail; i++ {
				buf[i] = buf[i-1]
			}
			tail--
		}
		buf[tail] = s
		tail++
		allS := strings.Join(buf[:tail], "")
		newS := m.replaceAll(allS)
		if newS != allS {
			out <- newS
			tail = 0
		}
	}
	// send the rest in buf
	out <- m.replaceAll(strings.Join(buf[:tail], ""))
}
```

上面的代码第 21~24 行对 `buf` 的循环移位可以替换成循环列表，那么这一步的时间复杂度就变成 $O(1)$ 。总的空间复杂度为 $O(m)$ 。时间复杂度体现在 `replaceAll` 函数中，总的时间复杂度为 $O(n*len(refs)*O(strings.RepalceAll))=O(n*len(refs)*m*L*L)=O(n*m*len(refs*L^2)$（这里只考虑 `Send` 函数）。然而，上面的方法是有些缺陷的，因为每次匹配替换的时候，整个 `buf` 都会被弹出来，因此如果最后一个 `s` 的后缀恰好是某个 `k` 的前缀，这样就会有问题，例如：

```
refs: {"ab": "1", "e", "2"}
inputs: ["aba", "b"]
output: 1ab
```

这个缺陷是可以弥补的，也就是我们只弹出匹配到的最后一个下标之前的字符串，并把剩余的字符串合并成一个字符串加入到 `buf` 中，但每次匹配替换都维护一个下标未免过于麻烦，并且需要保证 `prefix(k) != suffix(v)`。嗯，上面这个方法暴露了各种各样的问题。问题根源在于每次需要去遍历 `refs` 多次匹配替换，导致我们需要考虑这些边界case。 因此，我们能不能保证每次最多匹配替换一次？

## 方法二：维护字符

仔细分析上面的时间复杂度主要再 `replaceAll` 函数中，也就是查找是否存在子字符串 `k`。与上面维护字符串的方式不同，这里我们仅维护长度固定的字符，`buf` 的后缀是否能够和 `k` 匹配上，`map` 的查找时间复杂度为 $O(\log N)$ ，因此查找是否存在`k`  的时间复杂度就变成了 $O(\log(len(refs)))$。

```go
type refManager struct {
	refs map[string]string
}

func (m *refManager) replaceAll(s string) string {
	newS := s
	for k, v := range m.refs {
		newS = strings.ReplaceAll(newS, k, v)
	}
	return newS
}

func (m *refManager) Recv(in <-chan string, out chan string) {
	L := 4
	tail := 0
	buf := make([]byte, L)
	// collect len of all keys
	keyLensMap := make(map[int]struct{}, 1)
	for k := range m.refs {
		keyLensMap[len(k)] = struct{}{}
	}
	keyLens := make([]int, 0, len(keyLensMap))
	for l := range keyLensMap {
		keyLens = append(keyLens, l)
	}
	for s := range in {
		for _, c := range []byte(s) {
			// buf overflow, remove the first, and the first element cannot be the part of k
			if tail >= L {
				out <- string(buf[0])
				for i := 1; i < tail; i++ {
					buf[i] = buf[i-1]
				}
				tail--
			}
			buf[tail] = c
			for _, l := range keyLens {
				if l > tail {
					continue
				}
				if v, ok := m.refs[string(buf[tail-l:tail])]; ok {
					out <- v
					tail = tail - l
				}
			}
		}
	}
	// send the rest in buf
	out <- string(buf[:tail])
}
```

代码和第一个方法有点像，空间复杂度是一样的为 $O(m)$，时间复杂度为 $O(n*m*\log(len(refs)))$，时间复杂度确实是降下来了。并且每次我们只会替换 `buf` 的后缀，  但这个方法却不适合单独用在业务中，为什么？因为多字节码字会被截断，导致输出乱码。

因此在输出端，我们需要一个 `RuneStreamReader` 用于缓存输出字节，使得被截断的多字节码字重新组成成正常的码字。`RuneStreamReader` 实现两个接口 `io.Writer` 和 `Read() string` ，`Read() string` 贪心读取 `buffer` 中完整的多字节码字。由于 `RuneStreamReader` 是串联到输出端的，因此时间复杂度不会增大，并且方法二能用在通用场景下，对数据没有很严格的要求。

## 后面

其实在一开始做这个需求的时候，我思考的方向就是这两个策略，然后分析时间复杂度与实现的难度，选择了后者。但一开始没有考虑到多字节码字被截断的情况，后面在单测环节出现了乱码，然后分析出原因，在输出端加上了缓存队列，导致整个复杂度拔高了，后面打算切换到第一种方法。但是就在写这篇文章的时候，我发现了第一个方法有很多边界case其实是难以处理的，幸运的是业务逻辑我还没开始动，因此最终还是沿用了第二种方法，其实写博客也是相当于自行梳理了一遍。
