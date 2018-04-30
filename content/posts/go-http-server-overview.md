---
title: "Goのhttp serverの雰囲気を理解する"
date: 2018-04-29T00:03:43+09:00
draft: true
---

# Goのhttp serverの雰囲気を理解する
Goのhttpサーバの雰囲気を理解するための記事を書いた。  
<!--more-->

# tl; dr
- Goのhttpサーバの肝はHandler interfaceなので、理解できると雰囲気がわかる。  
- middlewareやrouterはHandler interfaceを上手く使うことで実現できる。
- middlewareは、Handler interfaceを満たす状態を保ったまま処理を付け足す  
（= decoratorパターン）
- routerは、Handler interfaceを別の実装に置き換える

# 始めてhttpサーバを立てるときの気持ち
ググってみて、以下のようなコードを書くはず。
```go
package main

import (
	"fmt"
	"net/http"
)

type fuga int

func (f *fuga) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "fuga type: %d", *f)
}

func main() {
	http.HandleFunc("/hoge", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "Hello world")
	})

	f := fuga(1)
	http.Handle("/fuga", &f)

	if err := http.ListenAndServe(":8080", nil); err != nil {
		panic(err)
	}
}
```
HandleFuncやHandleメソッドを使ってhandlerを登録することができるのだな、handlerは特定のURLにマッピングされ、マッチするURLへのアクセス時にhandlerが実行されるのだな、と直感的に理解できると思う。

# 本格的に使おうとするときの気持ち
登録するhandlerの数が増えたり、より複雑になってくると、もっと上手くやれないものかと思うはず。例えば以下のようなもの。 

- 様々なhandler間で共通の処理を上手く書けないか  
	- ロギング
	- 認証
	- Headerの付加や
	- Header情報に基づく共通処理、など
- もっと便利にroutingの設定をできないか  
	- 個別のHTTPメソッドに対してhandlerを登録したい
	- path parameterを簡単にハンドリングしたい、など
- 標準ライブラリのhttpサーバよりも実行効率のよいhttpサーバを作りたい

サードパーティのライブラリやフレームワークを使えば、これらの要望を満たすことができる。では、サードパーティのライブラリやフレームワークは、何を、どのように実装しているのだろうか。それらの雰囲気を理解するために、標準のhttpサーバの挙動の雰囲気を追ってみる。

# 標準ライブラリのhttpサーバの挙動
Goのソースコードから、雰囲気を汲み取ってみる。  
まずは、`http.ListenAndServe(":8080", nil)`の実行後の挙動を追う。
``` go
2966 // ListenAndServe always returns a non-nil error.
2967 func ListenAndServe(addr string, handler Handler) error {
2968     server := &Server{Addr: addr, Handler: handler}
2969     return server.ListenAndServe()
2970 }
```
`Server`インスタンスを初期化して、`ListenAndServe`メソッドを呼び出す。  
Serverインスタンスは、`Handler`インタフェースを持つ、これがすごく重要。  
具象型ではなく、インタフェースになっているのがポイント。
``` go
2697 // ListenAndServe listens on the TCP network address srv.Addr and then
2698 // calls Serve to handle requests on incoming connections.
2699 // Accepted connections are configured to enable TCP keep-alives.
2700 // If srv.Addr is blank, ":http" is used.
2701 // ListenAndServe always returns a non-nil error.
2702 func (srv *Server) ListenAndServe() error {
2703     addr := srv.Addr
2704     if addr == "" {
2705         addr = ":http"
2706     }
2707     ln, err := net.Listen("tcp", addr)
2708     if err != nil {
2709         return err
2710     }
2711     return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
2712 }
```
TCPサーバをListenしている。この後、`func (srv *Server) Serve(l net.Listener) error {}`や`func (c *conn) serve(ctx context.Context) {}`が順次実行され、TCPサーバとしての役割を全うするがここでは割愛する。（長いので）

注目したいのは、始めにインスタンス化した`Server`インスタンスが後続の処理に引き継がれていること。  

- `func (srv *Server) Serve(l net.Listener) error {}`のレシーバは`Server`インスタンスのポインタ  
- `func (c *conn) serve(ctx context.Context) {}`のレシーバは`conn`インスタンスのポインタで、`conn`はメンバとして`Server`のポインタを保持している

ソケットから情報を読み出したあとで、`Server`インスタンスは使われる。

```go
1830         serverHandler{c.server}.ServeHTTP(w, w.req)
```
`func (c *conn) serve(ctx context.Context) {}`の中のこの処理で、引き継がれた`Server`インスタンスが持つ`Handler`インタフェースを使って、dispatchが始まる。

```go
2560 func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
2561     handler := sh.srv.Handler
2562     if handler == nil {
2563         handler = DefaultServeMux
2564     }
2565     if req.RequestURI == "*" && req.Method == "OPTIONS" {
2566         handler = globalOptionsHandler{}
2567     }
2568     handler.ServeHTTP(rw, req)
2569 }
```

2561行目のHandlerはinterfaceである。このHandlerは、`http.ListenAndServe`関数の第２引数として指定されたHandlerである。
Handler interfaceを満たしていればどんな型であってもよいので、この部分以降は実装により挙動が変わることになる（！）  
ここからは`http.ListenAndServe`関数の第２引数を`nil`とした場合に使われるdefaultの`DefaultServeMux`の挙動を追っていく。
```go

```

# httpサーバの構成要素
`ServeHTTP(rw ResponseWriter, req *Request)`メソッドを満たすものは、Handlerインタフェースを満たすtypeである。
- 異なるコンテキストの中で実行されていた

図にまとめると、こんな雰囲気になる。




# 全てはHandler interface

つまり、Handler interfaceを満たすコンポーネントを開発して、それをライブラリ/フレームワークで置き換えればよい。

# ライブラリやフレームワークの雰囲気
