+++
title = "go modules(vgo)でlabstack/echo依存のプロジェクトをビルドする"
date = 2018-07-14T19:47:30+09:00
draft = false 
tags = ['golang']
+++

# go modules(vgo)でecho依存のプロジェクトをビルドする
go modules(`vgo`)で`labstack/echo`を使ったプロジェクトをビルドするときにやったことを書いた。
<!--more-->

## はじめに
go modulesが`go`コマンドにマージされ、いよいよgo 1.11のリリースで`go`コマンドから`vgo`として提案されたgo modulesが使えるようになるということで、試してみた.  

- [go modules have landed](https://groups.google.com/forum/#!topic/golang-dev/a5PqQuBljF4)

`vgo`に関する記事を軽く読み流したことがあり、`vgo`がどういうものか知っている + 詳細な導入手順の記事などは真剣に読んではいない、という状態で色々試したことを書く.  
なお、go 1.11を手元でビルドすれば`go`コマンドからgo modulesが使えるが、今回はプロトタイプの`vgo`コマンドで試した.  

- [Taking Go modules for a spin](https://dave.cheney.net/2018/07/14/taking-go-modules-for-a-spin)

`vgo`関連のブログポストやドキュメントへのリンクは、オフィシャルのwikiにまとまっている.

- [vgo・golang/go Wiki](https://github.com/golang/go/wiki/vgo) 

## vgoをインストール.
```go
$ go get -u golang.org/x/vgo 

$ vgo version
go version go1.10 darwin/amd64 go:2018-02-20.1
```

## サンプルプロジェクトのコード
```go
package main

import (
	"net/http"

	"github.com/labstack/echo"
	"github.com/labstack/echo/middleware"
)

func main() {
	e := echo.New()
	e.Use(middleware.Logger())

	e.GET("/hello", hello)

	e.Logger.Fatal(e.Start(":8080"))
}

func hello(c echo.Context) error {
	return c.String(http.StatusOK, "hello world!")
}
```

## 最初のビルド
新しく作るプロジェクトでかつvendorディレクトリを持たないので、ドキュメント([New Project Setup](https://github.com/golang/go/wiki/vgo-user-guide#new-project-setup))にしたがってOption 1の通りにコマンドを実行した。

```sh
$ cd $GOPATH/src/<project path>
$ touch go.mod
$ vgo build ./...
go: finding github.com/labstack/echo/middleware latest
go: finding github.com/labstack/echo v1.4.4
go: downloading github.com/labstack/echo v1.4.4
go: finding github.com/labstack/gommon/color latest
go: finding github.com/labstack/gommon v0.2.1
go: downloading github.com/labstack/gommon v0.2.1
go: finding gopkg.in/labstack/echo.v1 v1.4.4
go: downloading gopkg.in/labstack/echo.v1 v1.4.4
go: finding golang.org/x/net/context latest
go: finding golang.org/x/net latest
go: downloading golang.org/x/net v0.0.0-20180712202826-d0887baf81f4
go: finding golang.org/x/net/websocket latest
go: finding github.com/labstack/gommon/log latest
go: finding github.com/valyala/fasttemplate latest
go: downloading github.com/valyala/fasttemplate v0.0.0-20170224212429-dcecefd839c4
go: finding github.com/mattn/go-isatty v0.0.3
go: downloading github.com/mattn/go-isatty v0.0.3
go: finding github.com/mattn/go-colorable v0.0.9
go: downloading github.com/mattn/go-colorable v0.0.9
go: finding github.com/valyala/bytebufferpool latest
go: downloading github.com/valyala/bytebufferpool v0.0.0-20160817181652-e746df99fe4a
# github.com/himetani/vgo-echo-sample
./main.go:14:3: e.GET undefined (type *"github.com/labstack/echo".Echo has no field or method GET, but does have Get)
./main.go:16:10: e.Logger.Fatal undefined (type func() *"github.com/labstack/gommon/log".Logger has no field or method Fatal)
./main.go:16:18: e.Start undefined (type *"github.com/labstack/echo".Echo has no field or method Start)
```

ビルドは失敗した。


## 何が起きたか?
1, `go.mod`、`go.sum`への依存情報の書き込み.   
プロジェクト自体が依存するライブラリ以外に、依存するライブラリが依存するライブラリの情報も書き込まれていた.
依存するライブラリが依存するライブラリの依存情報は、`go.mod`の中では`// indirect`のコメントが書いてあった.

```sh
# go.mod
module github.com/himetani/vgo-echo-sample

require (
        github.com/labstack/echo v1.4.4
        github.com/labstack/gommon v0.2.1 // indirect
        github.com/mattn/go-colorable v0.0.9 // indirect
        github.com/mattn/go-isatty v0.0.3 // indirect
        github.com/valyala/bytebufferpool v0.0.0-20160817181652-e746df99fe4a // indirect
        github.com/valyala/fasttemplate v0.0.0-20170224212429-dcecefd839c4 // indirect
        golang.org/x/net v0.0.0-20180712202826-d0887baf81f4 // indirect
        gopkg.in/labstack/echo.v1 v1.4.4 // indirect
)
```
```sh
# go.sum
github.com/labstack/echo v1.4.4 h1:1bEiBNeGSUKxcPDGfZ/7IgdhJJZx8wV/pICJh4W2NJI=
github.com/labstack/echo v1.4.4/go.mod h1:0INS7j/VjnFxD4E2wkz67b8cVwCLbBmJyDaka6Cmk1s=
github.com/labstack/gommon v0.2.1 h1:C+I4NYknueQncqKYZQ34kHsLZJVeB5KwPUhnO0nmbpU=
github.com/labstack/gommon v0.2.1/go.mod h1:/tj9csK2iPSBvn+3NLM9e52usepMtrd5ilFYA+wQNJ4=
github.com/mattn/go-colorable v0.0.9 h1:UVL0vNpWh04HeJXV0KLcaT7r06gOH2l4OW6ddYRUIY4=
github.com/mattn/go-colorable v0.0.9/go.mod h1:9vuHe8Xs5qXnSaW/c/ABM9alt+Vo+STaOChaDxuIBZU=
github.com/mattn/go-isatty v0.0.3 h1:ns/ykhmWi7G9O+8a448SecJU3nSMBXJfqQkl0upE1jI=
github.com/mattn/go-isatty v0.0.3/go.mod h1:M+lRXTBqGeGNdLjl/ufCoiOlB5xdOkqRJdNxMWT7Zi4=
github.com/valyala/bytebufferpool v0.0.0-20160817181652-e746df99fe4a h1:AOcehBWpFhYPYw0ioDTppQzgI8pAAahVCiMSKTp9rbo=
github.com/valyala/bytebufferpool v0.0.0-20160817181652-e746df99fe4a/go.mod h1:6bBcMArwyJ5K/AmCkWv1jt77kVWyCJ6HpOuEn7z0Csc=
github.com/valyala/fasttemplate v0.0.0-20170224212429-dcecefd839c4 h1:gKMu1Bf6QINDnvyZuTaACm9ofY+PRh+5vFz4oxBZeF8=
github.com/valyala/fasttemplate v0.0.0-20170224212429-dcecefd839c4/go.mod h1:50wTf68f99/Zt14pr046Tgt3Lp2vLyFZKzbFXTOabXw=
golang.org/x/net v0.0.0-20180712202826-d0887baf81f4 h1:KDF3PK6A+dkI7c4O8QbMtJqcXE3LdNJFGZECIlifQOg=
golang.org/x/net v0.0.0-20180712202826-d0887baf81f4/go.mod h1:mL1N/T3taQHkDXs73rZJwtUhF3w3ftmwwsq0BUmARs4=
gopkg.in/labstack/echo.v1 v1.4.4 h1:7OfDGi3z6gFaNSepZ92pgetqmLsPJyBpTfHjTCxCQ2w=
gopkg.in/labstack/echo.v1 v1.4.4/go.mod h1:Om7rDkDgmunv8fIK0aJk20dpMc/wL3gduK2/MA77j9M=

```

2, 依存ライブラリのダウンロード.  
`$GOPATH/src/mod`配下に、依存ライブラリがダウンロードされた.ダウンロードされたファイルは全て読み取り専用ファイルだった.

```sh
$ tree -L 3 $GOPATH/src/mod
/$GOPATH/src/mod
├── cache
│   ├── download
│   │   ├── github.com
│   │   ├── golang.org
│   │   └── gopkg.in
│   └── vcs
│       ├── 0dab07b2bc76e1a16f634a5820d4273df978197197563ecdd7de44174100fdff
│       ├── 0dab07b2bc76e1a16f634a5820d4273df978197197563ecdd7de44174100fdff.info
│       ├── 13abe9aa4277b39be515dfbd2c238cd85cb821e0f839d89b85f0550029fd014c
│       ├── 13abe9aa4277b39be515dfbd2c238cd85cb821e0f839d89b85f0550029fd014c.info
│       ├── 30c9ff783c569486f16dd17edc1d5452d3cb911f2bf5cbd3ad3a9cd40cac2cc9
│       ├── 30c9ff783c569486f16dd17edc1d5452d3cb911f2bf5cbd3ad3a9cd40cac2cc9.info
│       ├── 4a22365141bc4eea5d5ac4a1395e653f2669485db75ef119e7bbec8e19b12a21
│       ├── 4a22365141bc4eea5d5ac4a1395e653f2669485db75ef119e7bbec8e19b12a21.info
│       ├── 7028097e3b6cce3023c34b7ceae3657ef3f2bbb25dec9b4362813d1fadd80297
│       ├── 7028097e3b6cce3023c34b7ceae3657ef3f2bbb25dec9b4362813d1fadd80297.info
│       ├── 81422602c8b2420147f0b1035e7fbd13b7e346a0fa3789a04112c4987ab7cd1a
│       ├── 81422602c8b2420147f0b1035e7fbd13b7e346a0fa3789a04112c4987ab7cd1a.info
│       ├── e031cefd20551fa656143db80048e31ec1392e078fe6d468f774bdf7eec417c5
│       ├── e031cefd20551fa656143db80048e31ec1392e078fe6d468f774bdf7eec417c5.info
│       ├── f7e99db597f4d2fe3e4509a9af308dace72a13292b505deb909cd0df29c1468a
│       └── f7e99db597f4d2fe3e4509a9af308dace72a13292b505deb909cd0df29c1468a.info
├── github.com
│   ├── labstack
│   │   ├── echo@v1.4.4
│   │   └── gommon@v0.2.1
│   ├── mattn
│   │   ├── go-colorable@v0.0.9
│   │   └── go-isatty@v0.0.3
│   └── valyala
│       ├── bytebufferpool@v0.0.0-20160817181652-e746df99fe4a
│       └── fasttemplate@v0.0.0-20170224212429-dcecefd839c4
├── golang.org
│   └── x
│       └── net@v0.0.0-20180712202826-d0887baf81f4
└── gopkg.in
    └── labstack
        └── echo.v1@v1.4.4
```

## ビルド失敗の原因
`github.com/labstack/echo`の最新のリリースは`v3`系(=3.3.5、2018/07/14付)であるにも関わらず、`v1.4.4`で依存解決されていたため. v1系のAPIが最新のAPIと互換性がなかったため、ビルドできなかった.

- [Releases・labstack/echo](https://github.com/labstack/echo/releases)

## 色々試してみた
直感ベースで色々思いつくことを試した.

## 失敗例その１
ソースコードのimportを`github.com/labstack/echo`から`github.com/labstack/echo/v3`に書き直した.
vgoにおいて、もっとも大きな変化の一つであるsemantic versioningでは、major versionの異なるパッケージは異なるimport pathで表す.
例えば、`github.com/foo/var`というパッケージがあったときに、このパッケージの`v2`系を使うためには`github.com/foo/var/v2`というimport pathを指定する.
ということで、その通りに書き直して試した.

```sh
# main.go

import (
	"net/http"

	"github.com/labstack/echo/v3"
	"github.com/labstack/echo/middleware/v3"
)

...
```

```sh
$ vgo build ./...
go: downloading github.com/labstack/echo/v3 v3.2.1
go: finding github.com/labstack/echo/middleware/v3 latest
go: finding github.com/labstack/echo/middleware latest
go: import "github.com/himetani/vgo-echo-sample" ->
        import "github.com/labstack/echo/middleware/v3": cannot find module providing package github.com/labstack/echo/middleware/v3
go: import "github.com/himetani/vgo-echo-sample" ->
        import "github.com/labstack/echo/v3": cannot find module providing package github.com/labstack/echo/v3
```

ビルドは失敗した.  

`github.com/labstack/echo`は、まだgo modulesに対応してなかったので、モジュールを解決できなかったぽい.
面白いなと思ったのが、  
`go: downloading github.com/labstack/echo/v3 v3.2.1`  
の部分.  
go modulesに対応していないというエラーが出ているものの、`v3.2.1`というバージョンが`v3`系の最新のバージョンだということは認識できてるぽい.

### 失敗例２
ソースコードのimportを`github.com/labstack/echo`から`gopkg.in/labstack/echo.v3`に書き直した.  

[gopkg.in - Stable APIs for the Go language](http://labix.org/gopkg.in)は、Russ Cox先生の一連の`vgo`の記事の中でも紹介されているサービスで、同じパッケージの異なるバージョンに対して、異なるURLを提供し、それぞれそのパッケージの異なるGitのコミットのURLにリダイレクトする.  

- [Go += Package Versioning (Go & Versioning, Part 1)](https://research.swtch.com/vgo-intro)

例えば、import pathに`gopkg.in/yaml.v1`を指定すると、`yaml`パッケージの`v1`系へのGitのコミットにリダイレクトされ、`gopkg.in/yaml.v2`を指定すると`v2`系にリダイレクトされる.
`github.com/labstack/echo`と書くと`v1`系で依存解決されてしまう、かつ、まだgo modules対応がされておらず`github.com/labstack/echo/v3`は使えないので、import pathを`gopkg.in/labstack/echo.v3`に書き直すということを試した.

サンプルコードを修正し、`go.mod`、`go.sum`を削除して、もう一度ビルドした.

```go
import (
	"net/http"

	"gopkg.in/labstack/echo.v3"
	"gopkg.in/labstack/echo.v3/middleware"
)

...

```

```sh
$ rm go.mod go.sum
$ touch go.mod
$ vgo build ./...
go: finding gopkg.in/labstack/echo.v3 v3.2.1
go: downloading gopkg.in/labstack/echo.v3 v3.2.1
go: finding github.com/labstack/gommon/bytes latest
go: finding github.com/dgrijalva/jwt-go v1.0.2
go: downloading github.com/dgrijalva/jwt-go v1.0.2
go: finding github.com/valyala/fasttemplate latest
go: finding github.com/labstack/gommon/color latest
go: finding github.com/labstack/gommon/random latest
go: finding golang.org/x/crypto/acme/autocert latest
go: finding golang.org/x/crypto/acme latest
go: finding golang.org/x/crypto latest
go: downloading golang.org/x/crypto v0.0.0-20180621125126-a49355c7e3f8
go: finding github.com/labstack/gommon/log latest
go: finding github.com/valyala/bytebufferpool latest
go: finding golang.org/x/net/context latest
go: finding golang.org/x/net latest
go: finding golang.org/x/net/websocket latest
# gopkg.in/labstack/echo.v3/middleware
../../../mod/gopkg.in/labstack/echo.v3@v3.2.1/middleware/jwt.go:34:10: undefined: jwt.Claims
```

ビルドは失敗した.

失敗した原因は、`gopkg.in/labstack/echo.v3/middleware`が依存する`github.com/dgrijalva/jwt-go`の最新のリリースは`v3`系(=3.2.0, 2018/07/14付)であるにも関わらず、`v1.0.2`で依存解決されていたため. 
v1系のAPIが最新のAPIと互換性がなかったため、ビルドできなかった.  

- [Releases・dgrijalva/jwt-go](https://github.com/dgrijalva/jwt-go/releases)

projectが依存するパッケージが依存するパッケージ(= indirect)をimportするときのURLを`gopkg.in`を使うように書き直すことはできないので、自分がビルドするプロジェクトのimport pathを書き直す方法だけではビルドは通せないとわかった.

### 失敗例３
失敗例２の状態で、`go.mod`の`replace`ディレクティブを使って、`github.com/...`で`v1`系で依存解決されているものを`gopkg.in`を使うように色々書き直した.

```sh
module github.com/himetani/vgo-echo-sample

require (
	github.com/dgrijalva/jwt-go v1.0.2 // indirect
	github.com/labstack/echo v1.4.4
	github.com/labstack/gommon v0.2.1 // indirect
	github.com/mattn/go-colorable v0.0.9 // indirect
	github.com/mattn/go-isatty v0.0.3 // indirect
	github.com/valyala/bytebufferpool v0.0.0-20160817181652-e746df99fe4a // indirect
	github.com/valyala/fasttemplate v0.0.0-20170224212429-dcecefd839c4 // indirect
	golang.org/x/crypto v0.0.0-20180621125126-a49355c7e3f8 // indirect
	golang.org/x/net v0.0.0-20180712202826-d0887baf81f4 // indirect
	gopkg.in/labstack/echo.v1 v1.4.4 // indirect
	gopkg.in/labstack/echo.v3 v3.2.1
)

replace (
	github.com/dgrijalva/jwt-go v1.0.2 => gopkg.in/dgrijalva/jwt-go.v3 v3.2.0
	github.com/labstack/echo v1.4.4 => gopkg.in/labstack/echo.v3 v3.2.1
)
 ```

 ```sh
$ vgo build 
# github.com/himetani/vgo-echo-sample
./main.go:12:25: cannot use middleware.Logger() (type "github.com/labstack/echo".MiddlewareFunc) as type "gopkg.in/labstack/echo.v3".MiddlewareFunc in argument to e.Use
 ```

 依存解決はうまくいったようだが、ビルドができなかった。。。試しに`go build`してみたが、これでもビルドできなかったので、`vgo`うんぬんの話ではなくそもそも`gopkg.in`を使うとimport pathを解決できずに上手くビルドできないという問題がある.


## 成功例
`vgo get`コマンドを使って、`go.mod`の`github.com/labstack/echo`の依存するバージョンを`v3`系にあげた. `vgo get`を使うと、依存するモジュールのバージョンを変更することができた.

```sh
$ rm go.mod go.sum
$ touch go.mod
$ vgo build ./...                                                                                                                
go: finding github.com/labstack/echo/middleware latest
go: finding github.com/labstack/gommon/color latest
go: downloading gopkg.in/labstack/echo.v1 v1.4.4
go: finding golang.org/x/net/context latest
go: finding golang.org/x/net latest
go: finding golang.org/x/net/websocket latest
go: finding github.com/labstack/gommon/log latest
go: finding github.com/valyala/fasttemplate latest
go: finding github.com/valyala/bytebufferpool latest
# github.com/himetani/vgo-echo-sample
./main.go:14:3: e.GET undefined (type *"github.com/labstack/echo".Echo has no field or method GET, but does have Get)
./main.go:16:10: e.Logger.Fatal undefined (type func() *"github.com/labstack/gommon/log".Logger has no field or method Fatal)
./main.go:16:18: e.Start undefined (type *"github.com/labstack/echo".Echo has no field or method Start)

$ vgo get github.com/labstack/echo@v3.3.5
go: finding github.com/labstack/echo v3.3.5
go get github.com/labstack/echo@v3.3.5: unknown revision v3.3.5
```

`v3.3.5`はダウンロードできないようなので、さっきダウンロードできてた`v3.2.1`で試した.

```sh
vgo get github.com/labstack/echo@v3.2.1                                                                                        
go: finding github.com/labstack/echo v3.2.1
go: downloading github.com/labstack/echo v0.0.0-20170621150743-935a60782cbb
go: finding golang.org/x/crypto/acme/autocert latest
go: finding golang.org/x/crypto/acme latest
go: finding golang.org/x/crypto latest
```

`go.mod`を確認すると`github.com/labstack/echo`が更新されていた.

```sh
# go.mod

require (
        github.com/labstack/echo v0.0.0-20170621150743-935a60782cbb
		... 
)
```

```sh
$ vgo build ./...
go: finding github.com/labstack/echo v0.0.0-20170621150743-935a60782cbb
# github.com/labstack/echo/middleware
../../../mod/github.com/labstack/echo@v0.0.0-20170621150743-935a60782cbb/middleware/jwt.go:34:10: undefined: jwt.Claims
```

`github.com/dgrijalva/jwt-go`もバージョンをあげる.

```sh
$ vgo get github.com/dgrijalva/jwt-go@v3.2.0                                                                                     
go: finding github.com/dgrijalva/jwt-go v3.2.0
go: downloading github.com/dgrijalva/jwt-go v0.0.0-20180308231308-06ea1031745c
```

`go.mod`を確認すると`github.com/dgrijalva/jwt-go`が更新されていた.

```sh
# go.mod 

require (
        github.com/dgrijalva/jwt-go v0.0.0-20180308231308-06ea1031745c // indirect
		...
)
```

```sh
$ vgo build ./...                                                                                                                
go: finding github.com/dgrijalva/jwt-go v0.0.0-20180308231308-06ea1031745c
$ ls                                                                                                                             
go.mod          go.sum          main.go         vgo-echo-sample
```

これでようやくビルドが成功した.

## 気になって調べたこと
- なぜ`vgo`コマンドで`v3.3.5`が依存解決できなかったのか？  
=> `vgo`は`v`のprefixがついたタグのみ認識するため.
実は`github.com/labstack/echo`では、`v`がついているのが`v3.2.1`までで、それ以降のバージョンにはついていない.
- `go.mod`に書かれているバージョンが`v0.0.0-20180308231308-06ea1031745c`となっているがこれは何か？  
=> pseudo-versionと言われるもので、コミットが直接指定されている. pseudo-versionで指定されているコミットは、`v0.0.1`より小さいバージョンだとみなされる.

## まとめ
- `vgo`では、`v0`、`v1`のバージョンは`github.com/foo/bar`、`v2`系以降のバージョンは`github.com/foo/bar/v2`のようにモジュールの依存を指定する.
- go modulesに対応していないパッケージの`v2`系以降のバージョンを使うには`vgo get github.com/foo/bar@tag`とすることで`github.com/foo/bar`のままで利用できる. ただし、そのバージョンはpseudo-versionとなる.
- pseudo-versionは、例えそれが`v2`系以降を示すバージョンであったとしても、go modulesの中では`v2`系以降のバージョンとは認識されない. 特定のcommitが直接指定されて依存解決される. 
- 利用するパッケージがgo modulesに対応しておらず、`v2`系以降のバージョンを使いたい、もしくは利用するパッケージが依存するパッケージの依存の中にgo modulesに対応していないかつ`v2`系以降のバージョンの依存が含まれる場合、`vgo get`でいちいちpsuedo versionを指定しないとbuildできないので、めんどくさそう. (depやglideのプロジェクトを移行する場合は`vgo`がいい感じにやってくれるみたいなので、面倒くさいのは新規のプロジェクト)
