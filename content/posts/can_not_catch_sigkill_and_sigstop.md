+++
title = "SIGKILL、SIGSTOPはcatchできない. Golangでcatchしようとするコードを書いても、コンパイルエラーにもランタイムエラーにもならない"
date = 2018-08-05T18:03:09+09:00
draft  = true
tags = ["golang"]
+++

GolangでSIGKILLやSITGTOPをcatchしようとしてしまったときのことを書いた.

<!--more-->

# やらかしたこと　
Goで常駐プロセスを動かすときに、SIGNALをトラップしてgracefulにプロセスを止めることができるように以下のようなプログラムを書くことがあった.  

```go
package main

import (
	"context"
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	c := make(chan os.Signal)
	signal.Notify(c, syscall.SIGINT)

	ctx, cancel := context.WithCancel(context.Background())

	go func() {
		sig := <-c
		fmt.Printf("Got %s signal. Aborting...\n", sig)
		cancel()
	}()

	doSomething(ctx)
}

func doSomething(ctx context.Context) {
	defer fmt.Println("End of doSomething func")
	for {
		select {
		case <-time.Tick(1 * time.Second):
			fmt.Println("hello")
		case <-ctx.Done():
			return
		}
	}
}
```

このプログラムを停止するには、このプロセスにSIGINTシグナルを送ればよい.  
SIGKILLやSIGSTOPをでもgracefulに停止できるようにしようと思って以下のように変えてみたが、これは間違いだ.
コンパイルエラーもランタイムエラーも起きないが、シグナルをcatchすることはできない.

``` go
signal.Notify(c, syscall.SIGINT, syscall.SIGKILL, syscall.SIGSTOP)
```

## SIGKILL, SIGSTOPはcatchできない
そもそもSIGKILLとSITSTOPは、catchできないシグナルだった...   
[Signal (IPC)](https://en.wikipedia.org/wiki/Signal_(IPC)#SIGKILL)

Goのオフィシャルドキュメントにも明記してある.  
`The signals SIGKILL and SIGSTOP may not be caught by a program, and therefore cannot be affected by this package.`

[Package signal](https://golang.org/pkg/os/signal/)


上記のようなコードはコンパイルエラーにするべきだ、というissueが上がっていましたが、見送られたようです.  
[os/signal: Prevent developers from catching SIGKILL and SIGSTOP](https://github.com/golang/go/issues/9463)

理由は以下.

- `signal.Notify(c chan<- os.Signal, sig ...os.Signal)`のAPIとしては正しいのでコンパイルエラーにすべきでない
- このような問題の検出には、go vetが役に立つ
- Go 2だと改善する？
- ランタイムエラーとするにはerrorを返すようにAPIを変更するかpanicを起こす必要があるが、いずれも既存のコードの互換性が壊れるので厳しい

などなど

# まとめ

- SIGKILLとSIGSTOPは、そもそもcatchできない.
- GoでSIGKILLとSIGSTOPをcatchするコードは、コンパイルエラーにもランタイムエラーにもならないので間違って書かないように気をつけましょう.
