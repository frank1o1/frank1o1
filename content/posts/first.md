---
title: "GoLang面试题目解析"
date: 2022-03-09T10:27:30+08:00
draft: false
---

## 交替打印数字和字母

**问题描述**

使用两个 `goroutine` 交替打印序列，一个 `goroutine` 打印数字， 另外一个 `goroutine` 打印字母， 最终效果如下：

```go
12AB34CD56EF78GH910IJ1112KL1314MN1516OP1718QR1920ST2122UV2324WX2526YZ2728
```

```go
package main

import (
	"fmt"
)

func print(value interface{}, chans chan int8, types int8) {
	switch value.(type) {
	case int:
		fmt.Print(value)
		fmt.Print(value.(int) + 1)
	case rune:
		fmt.Print(string(value.(rune)))
		fmt.Print(string(value.(rune) + 1))
	}
	chans <- types
}

func main() {
	var num = 1
	var str = 'A'
	var printChan = make(chan int8, 1)
	printChan <- 1
	for {
		select {
		case types := <-printChan:
			if types == 1 {
				go print(num, printChan, 2)
				num = num + 2
			} else {
				go print(str, printChan, 1)
				str = str + 2
			}
			if str >= 'Z' {
				return
			}
		}
	}
}

```

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	letter, number := make(chan bool), make(chan bool)
	wait := sync.WaitGroup{}

	go func() {
		i := 1
		for {
			if <-number {
				fmt.Print(i)
				i++
				fmt.Print(i)
				i++
				letter <- true
			}
		}
	}()
	wait.Add(1)
	go func(wait *sync.WaitGroup) {
		i := 'A'
		for {
			if i >= 'Z' {
				wait.Done()
				return
			}

			if <-letter {
				fmt.Print(string(i))
				i++
				fmt.Print(string(i))
				i++
				number <- true
			}

		}
	}(&wait)
	number <- true
	wait.Wait()
}

```

## 判断字符串中字符是否全都不同

**问题描述**

请实现一个算法，确定一个字符串的所有字符【是否全都不同】。这里我们要求【不允许使用额外的存储结构】。 给定一个string，请返回一个bool值,true代表所有字符全都不同，false代表存在相同的字符。 保证字符串中的字符为【ASCII字符】。字符串的长度小于等于【3000】。

```go
package main

import (
	"fmt"
	"strings"
)

func isUniqueString(str string) bool {
	if len(str) > 3000 {
		return false
	}
	for _, v := range str {
		if v > 127 {
			return false
		}
		// method 1
		if strings.Count(str, string(v)) > 1 {
			return false
		}
		// method 2
		// if strings.Index(str,string(v)) != k {
		// 	return false
		// }
	}
	return true
}

func main() {
	var str = "uniqre"
	fmt.Print(isUniqueString(str))
}

```

## 翻转字符串

**问题描述**

请实现一个算法，在不使用【额外数据结构和储存空间】的情况下，翻转一个给定的字符串(可以使用单个过程变量)。

给定一个string，请返回一个string，为翻转后的字符串。保证字符串的长度小于等于5000。

```go
func reverseString(str string) string {
	strSlice := []rune(str)
	for index := range strSlice {
		if index >= len(strSlice)/2 {
			break
		}
		strSlice[index], strSlice[len(strSlice)-index-1] = strSlice[len(strSlice)-index-1], strSlice[index]
	}
	return string(strSlice)
}
```

## 判断两个给定的字符串排序后是否一致

**问题描述**

给定两个字符串，请编写程序，确定其中一个字符串的字符重新排列后，能否变成另一个字符串。 这里规定【大小写为不同字符】，且考虑字符串重点空格。给定一个string s1和一个string s2，请返回一个bool，代表两串是否重新排列后可相同。 保证两串的长度都小于等于5000。

```go
func bubbleSort(str string) string {
	strSlice := []rune(str)
	for i := 0; i < len(strSlice); i++ {
		for j := 0; j < len(strSlice)-i-1; j++ {
			if strSlice[j] > strSlice[j+1] {
				strSlice[j], strSlice[j+1] = strSlice[j+1], strSlice[j]
			}
		}
	}
	return string(strSlice)
}

func compareStr(str1, str2 string) bool {
	return str1 == str2
}
```

```go
func compareStr(str1, str2 string) bool {
	strSlice1 := []rune(str1)
	for _, value := range strSlice1 {
		if strings.Count(str1, string(value)) != strings.Count(str2, string(value)) {
			return false
		}
	}
	return true
}
```

## 字符串替换问题

**问题描述**

请编写一个方法，将字符串中的空格全部替换为“%20”。 假定该字符串有足够的空间存放新增的字符，并且知道字符串的真实长度(小于等于1000)，同时保证字符串由【大小写的英文字母组成】。 给定一个string为原始的串，返回替换后的string。

```go
func replaceStr(str string) string {
	strSlice := []rune(str)
	if len(strSlice) > 1000 {
		return str
	}
	for _, v := range strSlice {
		if unicode.IsLetter(v) {
			return str
		}
	}
	return strings.ReplaceAll(str, " ", "%20")
}
```

## 实现阻塞读且并发安全的map
**问题描述**
Go里面map如何实现key不存在，get操作等待，直到key存在或者超时，保证并发安全，并实现以下接口
```go
type sp interface {
	Save(key string, val interface{})
	Read(key string, timeout time.Duration) interface{}
}
```

```go
type Map struct {
	s map[string]*Store
	*sync.RWMutex
}

func (m *Map) Save(key string, val interface{}) {
	m.Lock()
	defer m.Unlock()
	if _, ok := m.s[key]; !ok {
		m.s[key] = &Store{
			ch:      make(chan struct{}),
			value:   val,
			isExist: true,
		}
		close(m.s[key].ch)
		return
	}
	m.s[key].value = val
	if !m.s[key].isExist {
		if m.s[key].ch != nil {
			close(m.s[key].ch)
			m.s[key].ch = nil
		}
	}
	return
}

func (m *Map) Read(key string, timeout time.Duration) interface{} {
	m.Lock()
	item, ok := m.s[key]
	if ok && item.isExist {
		m.Unlock()
		return item.value
	} else if !ok {
		m.s[key] = &Store{
			ch: make(chan struct{}),
		}
		m.Unlock()
		select {
		case <-m.s[key].ch:
			return m.s[key].value
		case <-time.After(timeout):
			fmt.Println("time out")
			return nil
		}
	} else {
		m.Unlock()
		select {
		case <-m.s[key].ch:
			return m.s[key].value
		case <-time.After(timeout):
			fmt.Println("time out")
			return nil
		}
	}

}

type Store struct {
	ch      chan struct{}
	value   interface{}
	isExist bool
}
```

