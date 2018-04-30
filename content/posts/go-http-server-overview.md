---
title: "Goのhttp serverの雰囲気を理解する"
date: 2018-04-29T00:03:43+09:00
draft: true
---

# Goのhttp serverの雰囲気を理解する
Goのhttpサーバの雰囲気を理解するための記事を書いた。  
<!--more-->

# tl; dr
- Goのhttpサーバの肝は`Handler`インタフェースなので、`Handler`インタフェースに注目すると雰囲気が掴みやすい。  
- `Handler`インタフェースが、ライブラリやフレームワークを実現するための拡張性、柔軟性を提供している。（すごい）

# Handlerインタフェースとは
```
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
第一引数にResponseWriter型、第二引数にRequestのポインタ型を取る`ServeHTTP`というメソッドを実装している型を扱うためのインタフェース。
Goのhttpサーバは、この`Handler`インタフェースを上手く使っている。  
以下の説明で、`Handler`インタフェースという文言は、標準パッケージの`http.Handler`を指しているものとする。

# 始めてhttpサーバを立てるとき
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
`HandleFunc`メソッドや`Handle`メソッドを使ってhandlerを登録することができるのだな、handlerは特定のURLにマッピングされ、マッチするURLへのアクセス時にhandlerが実行されるのだな、と直感的に理解できると思う。

# 本格的に使おうとするとき
登録するhandlerの数が増えたり、より複雑になってくると、もっと上手くやれないものかと思うはず。例えば以下のようなもの。 

- 様々なhandler間で共通の処理を上手く書けないか  
	- ロギング
	- 認証
	- Header情報に基づく共通処理、など
- もっと便利にroutingの設定をできないか  
	- 個別のHTTPメソッドに対してhandlerを登録したい
	- path parameterを簡単にハンドリングしたい、など

サードパーティのライブラリやフレームワークを使えば、これらの要望を満たすことができる。では、サードパーティのライブラリやフレームワークは、何を、どのように実装しているのだろうか。それらの雰囲気を理解するために、まずは標準のhttpサーバの挙動の雰囲気を追ってみる。

# 標準ライブラリのhttpサーバの挙動
Goのソースコードから、雰囲気を汲み取ってみる。長ったらしいので、退屈であれば[主要な構成要素とHandlerインタフェース](#anchor)まで読み飛ばしてもらってもいいと思う。
まずは、`http.ListenAndServe(":8080", nil)`の実行後の挙動を追う。
``` go
2966 // ListenAndServe always returns a non-nil error.
2967 func ListenAndServe(addr string, handler Handler) error {
2968     server := &Server{Addr: addr, Handler: handler}
2969     return server.ListenAndServe()
2970 }
```
`Server`を初期化して、初期化した`Server`の`ListenAndServe`メソッドを呼び出す。  
`Server`は`Handler`インタフェースを持つが、これがすごく重要。  
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
- `func (srv *Server) Serve(l net.Listener) error {}`のレシーバは`Server`のポインタ  
- `func (c *conn) serve(ctx context.Context) {}`のレシーバは`conn`のポインタで、`conn`はメンバとして`Server`のポインタを保持している  

なぜ`Server`インスタンスが引き継がれることが重要かというと、`Server`が持っている`Handler`が後の処理で使われるため。

```go
1830         serverHandler{c.server}.ServeHTTP(w, w.req)
```
`func (c *conn) serve(ctx context.Context) {}`の中のこの処理で、引き継がれた`Server`が持つ`Handler`インタフェースを使って、dispatchが始まる。

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

2561行目の`Handler`はinterfaceである。この`Handler`は、`http.ListenAndServe`関数の第２引数として指定された`Handler`である。
`Handler`インタフェースを満たしていればどんな型であってもよいので、以降は`Handler`の実装により挙動が変わることになる（！）  
ここからは`http.ListenAndServe`関数の第２引数を`nil`とした場合に使われるdefaultの`DefaultServeMux`の挙動を追っていく。
`DefaultServeMux`はServeMux型で、もちろんServeMux型はServeHTTPを実装しており、`Handler`インタフェースを満たす。

```go
2326 // ServeHTTP dispatches the request to the handler whose
2327 // pattern most closely matches the request URL.
2328 func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
2329     if r.RequestURI == "*" {
2330         if r.ProtoAtLeast(1, 1) {
2331             w.Header().Set("Connection", "close")
2332         }
2333         w.WriteHeader(StatusBadRequest)
2334         return
2335     }
2336     h, _ := mux.Handler(r)
2337     h.ServeHTTP(w, r)
2338 }
```
`mux.Handler(r)`で、事前に登録していたhandlerとURLのマッピング情報から実際にdispatchするhandlerを取得する。  

``` go
2133 type ServeMux struct {
2134     mu    sync.RWMutex
2135     m     map[string]muxEntry
2136     hosts bool // whether any patterns contain hostnames
2137 }
2138
2139 type muxEntry struct {
2140     h       Handler
2141     pattern string
2142 }
```

`ServeMux`型は、キーがURLパターン、バリューが`Handler`を持つ`muxEntry`、なmap構造を持っており、ここにhandlerとURLのマッピング情報を保持している。  
 実は`http.HandleFunc()`や`http.Handle()`では、この`ServeMux`型のmにURLパターンとhandlerのマッピング情報が登録されている。  

 最後に取得したhandlerが実行される。

<a name="anchor"></a>
# 主要な構成要素とHandlerインタフェース
ここまで、雑に標準のhttpサーバの挙動を追ったが、ごちゃごちゃしているので主要な構成要素について`Handler`インタフェースとの関わりという観点で整理する。

| 型                | `Handler`インタフェースとの関わり | 役割 |
| ----------------- |:-------------------------------:| -----:|
| `http.Server`      		| `Handler`インタフェースを持つ     | TCPサーバ。handlerのディスパッチが始まるエントリポイント。Request、Responseを生成した後、持っている`Handler`インタフェースの`ServeHTTP`メソッドを呼び出す。  |
| `http.ServeMux`  `http.DefaultServeMux`の型     	| `Handler`インタフェースを満たす。    `m map[string]muxEntry`に`Handler`が登録してある  | `Server`に呼び出される。URLにマッチするhandlerを`m map[string]muxEntry`から取得し、そのhandlerの`ServeHTTP`メソッドを呼び出す。 |
| `http.muxEntry`の`Handler` | `Handler`インタフェースを満たす   | `ServeMux`に呼び出される。`http.Handle()`や`http.HandleFunc`で登録される`Handler`インタフェースの正体がこれ。  |

リクエストを受けてからhandlerがdispatchされるまでに、異なるレイヤで異なる働きをするものが実行されるが、レイヤ間の処理の受け渡しに`Handler`インタフェースが使われていることがわかる。 

つまり、ライブラリやフレームワークを実現するためには、`Handler`を持つ、もしくは`Handler`を満たすコンポーネントを開発してどこかのレイヤの実装を置き換える、または拡張すればよい。  

なので、フレームワークやライブラリが何をやっているか知りたいときは、`ServeHTTP`でgrepしてみるとよい。`ServeHTTP`メソッドを読んでみて、どのレイヤの処理を置き換えたり拡張したりしているのか考えると雰囲気が掴みやすい。

# ライブラリやフレームワークの雰囲気
## middleware
middlewareは、前章の`muxEntry`の`Handler`のレイヤの処理の拡張を行うことで、様々なhandler間で共通の処理を上手く書くことできる。
具体的にどう実装するか、は以下の記事に詳しい説明がある。  

[Middlewares in Go: Best practices and examples](https://www.nicolasmerouze.com/middlewares-golang-best-practices-examples/)  
[Making and Using HTTP Middleware](http://www.alexedwards.net/blog/making-and-using-middleware)  
[Writing HTTP Middleware in Go](https://justinas.org/writing-http-middleware-in-go/)  

`Handler`インタフェースを利用した、Decoratorパターンで実現できる。

サードパーティだと[gorilla/handlers](https://github.com/gorilla/handlers)のようなものがある。

## router
routerは、前章の`ServeMux`のレイヤの処理の実装を別のコンポーネントに置き換える。
[gorilla/mux](https://github.com/gorilla/mux)や[julienschmidt/httprouter](https://github.com/julienschmidt/httprouter)のようなものがある。`ServeMux`レイヤの処理を置き換えるので、ライブラリ独自のroutingを保持するデータ構造を持っていて、`ServeHTTP`メソッドの中で独自のroutingロジックを実装している。ここのroutingのロジックの実装をいい感じにすることで、個別のHTTPメソッドに対してhandlerを登録したり、path parameterのハンドリングができるようになる。  

ちなみに、`ServeMux`のレイヤの処理の実装を置き換えるので、それ以降の`muxEntry`の`Handler`レイヤの処理は必ずしも`Handler`インタフェースを満たしている必要はない。ライブラリ側でシグネチャを変えることもできるし、もちろん変えなくてもよい。

## その他
有名なフレームワークに、[labstack/echo](https://github.com/labstack/echo)がある。このフレームワークでは、エントリポイントである`http.Server`をラップした`Echo`というコンポーネントで全ての処理を置き換えている。
```go
Echo struct {
		stdLogger        *stdLog.Logger
		colorer          *color.Color
		premiddleware    []MiddlewareFunc
		middleware       []MiddlewareFunc
		maxParam         *int
		router           *Router
		notFoundHandler  HandlerFunc
		pool             sync.Pool
		Server           *http.Server
		TLSServer        *http.Server
		Listener         net.Listener
		TLSListener      net.Listener
		AutoTLSManager   autocert.Manager
		DisableHTTP2     bool
		Debug            bool
		HideBanner       bool
		HidePort         bool
		HTTPErrorHandler HTTPErrorHandler
		Binder           Binder
		Validator        Validator
		Renderer         Renderer
		Logger           Logger
	}
```

TCPサーバとしての`Server`型のみ利用しており、`Server`型から`http.ServeMux`レイヤを呼び出すところのみ`Handler`インタフェースを使っているが、それ以外のところでは独自のシグネチャを採用している。

以下は、echoのページにあるサンプル。
```
package main

import (
	"net/http"

	"github.com/labstack/echo"
	"github.com/labstack/echo/middleware"
)

func main() {
	// Echo instance
	e := echo.New()

	// Middleware
	e.Use(middleware.Logger())
	e.Use(middleware.Recover())

	// Routes
	e.GET("/", hello)

	// Start server
	e.Logger.Fatal(e.Start(":1323"))
}

// Handler
func hello(c echo.Context) error {
	return c.String(http.StatusOK, "Hello, World!")
}
```

httpサーバの起動は`e.Start(":1323")`という独自の実装となっている。また、echoへのhandlerの登録は、httpパッケージの`Handler`インタフェースではなく、独自のシグネチャとなっていることがわかる。

# まとめ
- Goでのhttpサーバの肝は`Handler`インタフェースなので、`Handler`インタフェースを中心に考えると雰囲気が掴みやすい。
- `Handler`インタフェースの実装（=`ServeHTTP`）を追えば、各ライブラリやフレームワークが何をやっているか、雰囲気を掴める。
- `Handler`というシンプルなインタフェースだけで拡張性や柔軟性をもたらしていて、すごい。

# 参考
- [Middlewares in Go: Best practices and examples](https://www.nicolasmerouze.com/middlewares-golang-best-practices-examples/)  
- [Making and Using HTTP Middleware](http://www.alexedwards.net/blog/making-and-using-middleware)  
- [Writing HTTP Middleware in Go](https://justinas.org/writing-http-middleware-in-go/)  
- [gorilla/handlers](https://github.com/gorilla/handlers)
- [gorilla/mux](https://github.com/gorilla/mux)
- [julienschmidt/httprouter](https://github.com/julienschmidt/httprouter)
- [labstack/echo](https://github.com/labstack/echo)
