+++
title = "Vimでsimplenoteを編集する"
date = "2016-02-16T21:10:22+09:00"
tags = ['vim', 'simplenote']
+++


[simplenote](https://simplenote.com)というサービスをVimから使うための設定について書きました。  

<!--more-->

# Simplenoteとは
- Evernoteのように使えるクラウドサービス。
- リッチな機能はなく、管理対象はテキスト(markdown)のみ。
- markdownに対応

# やったこと
[simplenote.vim](https://github.com/mrtazz/simplenote.vim)を使う。

# 手順

- simplenote.vimのインストール

- simpletenote.vimでアカウントの設定
.vimrcにsimplenoteのアカウント情報を記します。

```sh  
#.vimrc
NeoBundle 'mrtazz/simplenote.vim'

let g:SimplenoteUsername = "XXX"
let g:SimplenotePassword = "XXX"
```
アカウント設定は.simplenotercファイルに書いて、.vimrcから.simplenotercを呼び出すようにすると、設定を外部ファイルに切り出せる。
```sh
#.simplenoterc

let g:SimplenoteUsername = "XXX"
let g:SimplenotePassword = "XXX"
```

```sh
#.vimrc

NeoBundle 'mrtazz/simplenote.vim'

source .simplenoterc
```
- vimでよしなに扱うための設定追加  

```sh
#.simplenoterc
let g:SimplenoteUsername = "XXX"
let g:SimplenotePassword = "XXX"

let g:SimplenoteFiletype = "markdown" 'markdownファイルとして読み込む
let g:SimplenoteVertical = 1
```

:Simplenote -nで新しいメモを作成したり、:Simplenote -lでメモの一覧を見てメモを選んで編集、などの基本的な操作が行えるようになります。  

最後に、おそらくmemoとtodoという名前のメモをたくさん使うことになるので、vimのカスタムコマンドの設定をしました。  
さらに、shellからそのままmemoやtodoを呼び出せるようにaliasの設定を行いました。  
ついでに、メモの一覧をnote、新しいメモをnewを実行すると呼び出せるように設定しました。
```sh
#.simplenoterc
let g:SimplenoteUsername = "XXX"
let g:SimplenotePassword = "XXX"

let g:SimplenoteFiletype = "markdown"
let g:SimplenoteVertical = 1

command Todo Simplenote -o XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
command Memo Simplenote -o XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

XXXXの部分には、メモのkeyを記述します。  
メモのキーは、:Simplenote -Vで確認できます。

```sh
#.bashrc of .zshrc
alias note='vim -c "Simplenote -l"'
alias todo='vim -c Todo -c on'
alias memo='vim -c Memo -c on'
alias new='vim -c "Simplenote -n"'
```

TodoコマンドやMemoコマンドで開くと、ウィンドウが二つ開いてしまうので、onコマンドでウィンドウを一つになるように設定しています。
