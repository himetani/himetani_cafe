---
title: "for文の中でmethodをgoroutine化する"
date: 2018-03-30T22:41:19+09:00
draft: false
---

# for文の中でmethodをgoroutine化する

こんなコードを書くと、予期しない結果となる。

<!--more-->

```go
  1 package main
  2
  3 import (
  4     "fmt"
  5     "sync"
  6 )
  7
  8 type Task struct {
  9     name string
 10     data int32
 11 }
 12
 13 func (t *Task) PrintData() {
 14     fmt.Println(t.name, ":", t.data)
 15 }
 16
 17 func main() {
 18     var wg sync.WaitGroup
 19
 20     tasks := []Task{{"hello", 1}, {"world", 2}}
 21     wg.Add(len(tasks))
 22
 23     for _, task := range tasks {
 24         go func() {
 25             defer wg.Done()
 26             task.PrintData()
 27         }()
 28     }
 29     wg.Wait()
 30 }
```

実行結果
```sh
% go run main.go
world : 2
world : 2
```


goroutine化をしない同じようなコードだと、期待通りに動く。
```go
  1 package main
  2
  3 import "fmt"
  4
  5 type Task struct {
  6     name string
  7     data int32
  8 }
  9
 10 func (t *Task) PrintData() {
 11     fmt.Println(t.name, ":", t.data)
 12 }
 13
 14 func main() {
 15     tasks := []Task{{"hello", 1}, {"world", 2}}
 16
 17     for _, task := range tasks {
 18         task.PrintData()
 19     }
 20 }
```

実行結果
```sh
% go run main.go
hello : 1
world : 2
```

# 原因
- for文の中の各iteration間で、共通のインスタンスが使われるため

一つ目と二つ目のgoroutine間でtask変数のインスタンスは共有されている、かつ1回目のiterationで生成されたgoroutineはforループが終了した後に実行されるため、すでにtaskインスタンスの内容は、forループの最後のiteration時の内容に書き換わっている為に起こる。  

```go
  1 package main
  2
  3 import (
  4     "fmt"
  5     "sync"
  6 )
  7
  8 type Task struct {
  9     name string
 10     data int32
 11 }
 12
 13 func (t Task) PrintData() {
 14     fmt.Printf("Address of receiver: %x\n", t)
 15     fmt.Println(t.name, ":", t.data)
 16 }
 17
 18 func main() {
 19     var wg sync.WaitGroup
 20
 21     tasks := []Task{{"hello", 1}, {"world", 2}}
 22     wg.Add(len(tasks))
 23
 24     for _, task := range tasks {
 25         fmt.Printf("Address inside for-loop: %x\n", task)
 26
 27         go func() {
 28             defer wg.Done()
 29             fmt.Printf("Address inside goroutine : %x\n", task)
 30             task.PrintData()
 31         }()
 32     }
 33
 34     wg.Wait()
 35 }
 ```

実行結果
```sh
% go run main.go
Address inside for-loop: {68656c6c6f 1}
Address inside for-loop: {776f726c64 2}
Address inside goroutine : {776f726c64 2}
Address of receiver: {776f726c64 2}
world : 2
Address inside goroutine : {776f726c64 2}
Address of receiver: {776f726c64 2}
world : 2
```

# 対策
生成されるgoroutineで同一の変数インスタンスを共有することを防ぐ。

## 1. closureに値をbindする
```go
27         go func(t Task) {
28             defer wg.Done()
29             t.PrintData()
30         }(task)
```

## 2. for-loopの中で新しい変数を作る
```go
24     for _, task := range tasks {
25         task := task
26
27         go func() {
28             defer wg.Done()
29             task.PrintData()
30         }()
31     }
```

# 参考
- [What happens with closures running as goroutines?](https://golang.org/doc/faq#closures_and_goroutines)
- [How to use a method as a goroutine function](https://stackoverflow.com/questions/36121984/how-to-use-a-method-as-a-goroutine-function)
