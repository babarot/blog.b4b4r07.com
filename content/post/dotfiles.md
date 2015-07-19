+++
date = "2015-07-16T19:02:20+09:00"
tags = ["dotfiles", "cli"]
title = "優れた dotfiles を設計する"

+++

[![](https://raw.githubusercontent.com/b4b4r07/screenshots/master/dotfiles/title.png)][dotfiles]

OS のクリーンインストールは面倒くさい． アプリケーションをいちいちダウンロードしてきて，普段の勝手と同じになるように設定する必要がある．CLI においても同じで，設定ファイルをいちいち書いたり，普段どんなプラグインを使っていたかを思い出してダウンロードするのは面倒だ．
よくあるのは `.vimrc` などの設定ファイルを Dropbox や GitHub に置いておいて，環境を作り直したときにコピーする手法だ．
dotfiles はその手法の延長線上にあって，より便利に高速化・自動化した方法だ．

dotfiles とは UNIX 系の OS でいう設定ファイルのことで，ファイル名がドット `.` から始まることからそう呼ばれている．

# TL;DR

HTTP 経由でインストールできる dotfiles をつくって 1 分で環境構築を終わらせる．

# Getting Started

<p align="center">
  <a href="http://b4b4r07.com/dotfiles">
  <img width="35%" src="https://raw.githubusercontent.com/b4b4r07/screenshots/master/dotfiles/symbol.png">
  </a>
</p>

dotfiles を始めるのはとても簡単だ．GitHub アカウントがあるなら，「dotfiles」の名前でリポジトリを作って自分のドットファイルを転送するだけ．別に dotfiles という名前である必要はないが，GitHub をウォッチすればこの名前でホスティングしているケースが圧倒的に多い（極稀に config や rcfiles といった名前も見かける）．

dotfiles というのは GitHub にアップロードしただけじゃうまく使えない．ホームディレクトリにコピーする必要があるのだが，環境を変えるごとに

```console
$ cp dotfiles/.vimrc ~/.vimrc
```

とするのはとても面倒で，しかもドットファイルがたくさんあるなら尚更．まずはこれを自動化しよう．

簡単なシェルスクリプトを書くだけでいい．Ruby（Rakefile）や Lua など他のスクリプト言語で書いている人も見かけるが，基本的に環境を構築するときは何もインストールされていない状態なので，これらの高級言語や枯れていない技術などは使えない可能性がある．シェルスクリプトや make はこういった心配がないので重宝する．

```bash
#...

for f in .??*
do
    [ "$f" = ".git" ] && continue

    ln -snfv "$f" "$HOME"/"$f"
done
```

これがコピー・リンク用のスクリプトで，いわゆるインストールスクリプト（ *installation script* ）である．ドットファイルを列挙（`.??*`）して `ln` を実行している．コピーではなくシンボリックリンクなのは，変更の同期を楽にするためである．設定を変更したり改修したりしたときに，よしなに dotfiles へ同期されるので，あとは `git push` をするだけという環境までもっていける．

`.git` や `.DS_Store` などの不要なドットファイルは実行されないようにすればよい．

# HTTP 経由でインストールする

<p align="center">
  <a href="http://b4b4r07.com/dotfiles">
  <img width="35%" src="https://raw.githubusercontent.com/b4b4r07/screenshots/master/dotfiles/http.png">
  </a>
</p>

前項で dotfiles の基本は完成した．あとは環境が壊れたり新しくなったときに，

```console
$ git clone htttp://github.com/"${username}"/dotfile.git
$ cd dotfiles
$ ./install.sh
```

を実行するだけである．しかし，これでは少し問題が残っている．

- コマンドを 3 つ打たなければならない
- git がない場合

特に後者は面倒で，git がないと git のインストールからスタートになる．一般ユーザなどで git をインストールできないなどの場合は更に厄介で，GitHub にアップされている dotfiles の tarball や zipball の URL を取得して，`curl` や `wget` を使う必要がある．

ここらへんの障害を吸収したスクリプトがこれ．

```bash
#...

DOTPATH=~/.dotfiles

# git が使えるなら git
if has "git"; then
    git clone --recursive "$GITHUB_URL" "$DOTPATH"

# 使えない場合は curl か wget を使用する
elif has "curl" || has "wget"; then
    tarball="https://github.com/b4b4r07/dotfiles/archive/master.tar.gz"
    
    # どっちかでダウンロードして，tar に流す
    if has "curl"; then
        curl -L "$tarball"

    elif has "wget"; then
        wget -O - "$tarball"

    fi | tar xv -
    
    # 解凍したら，DOTPATH に置く
    mv -f dotfiles-master "$DOTPATH"

else
    die "curl or wget required"
fi

cd ~/.dotfiles
if [ $? -ne 0 ]; then
    die "not found: $DOTPATH"
fi

# 移動できたらリンクを実行する
for f in .??*
do
    [ "$f" = ".git" ] && continue

    ln -snfv "$DOTPATH/$f" "$HOME/$f"
done
```

長いようにみえるが，律儀に条件分岐して実行するだけなので意外と簡単だ．あとはこれを `install.sh` といった分かりやすいファイル名にしてアップロードする．

さて，`curl` でこれにアクセスするが，そのままだと HTML の構造ごと見えてしまうので github.com のサブドメインに raw を付けて実行する

```console
$ curl -L raw.github.com/"${username}"/dotfiles/"${branch:-master}"/install.sh
```

リダイレクトが発生するので `-L` オプションはマストになる．こうすると，先ほどの `install.sh` が表示されると思う．あとはこれをシェルに渡すだけだ．

```console
$ curl -L raw.github.com/"${username}"/dotfiles/master/install.sh | bash
```

これでシェル環境の構築が完了したのではなかろうか．ワンコマンドで済み，依存するツールを最小限にすることができた．raw 付きの URL も覚えられなくはない長さなので，すぐさまこれをタイプするだけで OK だ．

しかし，リンクが切れたとかで，再度 `install.sh` を実行するときや，HTTP 経由ではなくローカルから実行するには少し工夫が必要になる．このままだと，dotfiles のインストールから再開されてしまうからだ．

# deploy と initialize

*deploy* と *initialize* については，下の記事で解説したがもう一度おさらいしようと思う．

- [最強の dotfiles 駆動開発と GitHub で管理する運用方法](http://qiita.com/b4b4r07/items/b70178e021bef12cd4a2)

***deploy*** **とは**

> *deploy* とは，dotfiles にあるドットファイルをホームディレクトリに展開することを指す．便宜的にそう読んでいるだけで，その実態はコピーであったりシンボリックリンクを張ることをいっている．

***initialize*** **とは**

> *initialize* とは，環境を再現するのに必要なソフトウェアをインストールしたり，プラグインのダウンロード・セットアップやディレクトリ名を英語化したりなどの最後の仕上げ部分を指す．アップデートを除いて一回きりの設定なので便宜上こう呼ぶ

これらを一緒くたにしてしまっているインストールスクリプト（ *installation script* ）は設計上よろしくない．何故かと言うと，その説明にもある通り，*initialize* はセットアップ時の一回きりしか実行されないからだ．一方で *deploy* は何度も実行する場面がある．

- リンクが切れた時
- リンクされたファイルを削除した時
- 新しいファイルを dotfiles に追加した時
- など

このときに，*initialize* を動き出すとはっきりいって面倒で，Ctrl-C で中断したりリンク張るためにスクリプトからリンクを実行している部分を切り出して別ファイルで実行したりしなければならない．

## 設定例

例えば処理部分で切り分けて，オプションやサブコマンドで対応したりする．

```bash
#!/bin/bash

deploy() {
	#...
	echo "deploy"
}

initalize() {
	#...
	echo "init"
}

if [ "$1" = "deploy" -o "$1" = "d" ]; then
	deploy
elif [ "$1" = "init" -o "$1" = "i" ; then
	initalize
fi
```

オプションにすると `curl` でインストールするときに少しわかりづらい記述になってしまう．

```console
$ curl -L dot.example.com | bash -s -d
```

# b4b4r07/dotfiles

[![](https://raw.githubusercontent.com/b4b4r07/screenshots/master/dotfiles/logo.png)][dotfiles]

おおまかに優れた dotfiles の設計について説いた．次は筆者のリポジトリを例に見ていこうと思う．

- [b4b4r07/dotfiles ❤ GitHub](https://github.com/b4b4r07/dotfiles) [![](https://img.shields.io/github/stars/b4b4r07/dotfiles.svg?style=flat-square)](https://github.com/b4b4r07/dotfiles "b4b4r07/dotfiles")

> Testing my dotfiles repo on OS X to get my work environment ready in just a few moments. #VIM + #ZSH + #TMUX = Best Developer Environment [http://b4b4r07.com/dotfiles](http://b4b4r07.com/dotfiles)

筆者の場合，インストールスクリプト（ *installation script* ）の他に，後述するが make を使っている．make を使うのは環境依存性の排除を重視しているからだ．dotfiles のセットアップ時は，環境が整っていない状態なのでなるべくツールの依存性を少なくしなければならない．make であれば、だいたいの Unix ライクシステムで利用できる．環境依存性を少なくするベストプラクティスは Bourne Shell，make を使うことだ．

## インストール

`curl` でインストールを開始する．`wget` でも良い．

```console
$ bash -c "$(curl -L dot.b4b4r07.com)"
```

上の方法でなく，`curl -L dot.b4b4r07.com | sh` でもいいが，これだとサブシェル上でインストールが開始される．この dotfiles のインストールスクリプト（ *installation script* ）はシェルの再起動も自動化しているため，それを有効化するためにはカレントシェルで実行する必要がある．

このワンライナーにより，

1. dotfiles をダウンロードする．`git clone` で引っ張ってくるが，git がない場合は `curl` または `wget` を使う
2. 次に `make deploy` を実行する（やっていることは，各ドットファイルをホームディレクトリにリンクする）

ドットファイルは，ダウンロードしたディレクトリを起点に make によってリンクされる．また，そのディレクトリをそれ以後，そのユーザの dotfiles リポジトリとして扱う．そのパスは `$DOTPATH` で管理しているので，変更したい場合は，

```console
$ DOTPATH=/path/to/dotfiles curl -L ...
```

として実行する（ただし事前に export されている必要がある）．デフォルトの `$DOTPATH` は `~/.dotfiles`．

引数に `init` を渡すと，`make init` も実行する．

```console
$ bash -c "$(curl -L dot.b4b4r07.com)" -s init
```

アップデートも簡単で，

```console
$ make update
```

とすると，すぐさま最新版にしてくれる．make を使ったことで処理の切り分けが簡単になった．

## ロジック

[![](https://raw.githubusercontent.com/b4b4r07/screenshots/master/dotfiles/components_ja.png)](https://raw.githubusercontent.com/b4b4r07/screenshots/master/dotfiles/components_ja.png)

- `curl` が参照するのは GitHub にホストされた `etc/install` の raw ファイル
- `etc/install` はインストールスクリプト（ *installation script* ）であると同時に，それ自体がライブラリになっており、呼び出し方によってライブラリかインストールスクリプト（*installation script*）かを決めている
- dotfiles に同梱されている多くのコマンドやスクリプトは，このライブラリを参照している
- そのパス解決に `$DOTPATH` を使用する
- `$DOTPATH` はホームディレクトリにリンクされたドットファイル `.path` によって決定する

## DOTPATH

dotfiles に関してたった一つだけ，専用の環境変数 DOTPATH を設けている．それは dotfiles ディレクトリのパスを知る変数で，後述する vital ライブラリや init スクリプトのパス解決に使われる．

```bash
#!/bin/bash

. "$DOTPATH"/etc/lib/vital.sh

#...
```

`.path` というドットファイルをホームディレクトリにリンクすることで，dotfiles のパスを辿っている．コマンド化させた `dotpath` を用意しているので，実行するだけで簡単に取得できる．

```console
$ dotpath
/home/b4b4r07/.dotfiles
```

## make

この dotfiles では，make がすべての起点になる．

- `make deploy`: ドットファイルをリンクする
- `make init`: 環境構築をする
- `make list`: リンクされるドットファイルをリストする
- `make test`: dotfiles を検証する
- `make clean`: dotfiles とドットファイルを削除する

ユーザはそれ以外のディレクトリを覗く必要もないし，それ以外のファイルを実行する必要もない．

### deploy

*deploy* とは，ドットファイルをホームディレクトリにリンクすることをいう．`.bashrc` や `.vimrc` などのドットファイル（ファイル名の頭に `.` が付く）はホームディレクトリにあると各種アプリケーションが読み込むような慣習があるためだ．

```make
EXCLUSIONS := .DS_Store .git .gitmodules .travis.yml
CANDIDATES := $(wildcard .??*) bin
DOTFILES   := $(filter-out $(EXCLUSIONS), $(CANDIDATES))
DOTPATH    := $(realpath $(dir $(lastword $(MAKEFILE_LIST))))

deploy:
	@$(foreach val, $(DOTFILES), ln -sfnv $(abspath $(val)) $(HOME)/$(val);)
```

### init

*initialize* とは，各種アプリケーションの設定ファイル**以外**の環境設定やその他をいう．例えば，Vim では `.vimrc` で行う設定以外にプラグインのダウンロードという作業が必要である．また，普段使うソフトウェアのインストールやローカライズなど，環境構築で欠かせないプロセスをプログラム化したのがこのステップである．

```make
init:
	@DOTPATH=$(DOTPATH) bash $(DOTPATH)/etc/init/init.sh
```

`make init` は `$DOTPATH/etc/init/init.sh` を実行するだけである．では，そのシェルスクリプトは何をしているのかというと，

```bash
#...

for i in "$DOTPATH"/etc/init/"$(get_os)"/*[^init].sh
do
    if [ -f "$i" ]; then
        e_arrow "$(basename "$i")"; bash "$i"
    else
        continue
    fi
done

#...
```

実行しているプラットフォームで必要なプロセスを記述したシェルスクリプトを呼び出している．こうすることで一括した実行が可能になる．また，*deploy* のときのように make で書けなくないが，テスタブルにする必要があるためシェルスクリプトに書き，それを make で呼ぶ形式になっている．

### test

dotfiles が正しくインストールできるか，付随するシェルスクリプトが正しく動作するかなどをチェックする．

```console
$ make test
 ✔ deploying dot files...OK
 ✔ linking valid paths...OK
 ✖ /Users/b4b4r07/.dotfiles/etc/test/redirect_test.sh: 17: unit1
 ➜ check shellcheck...
     ✔ /Users/b4b4r07/.dotfiles/etc/init/init.sh...OK
     ✔ /Users/b4b4r07/.dotfiles/etc/init/common/pygments.sh...OK
     ✔ /Users/b4b4r07/.dotfiles/etc/init/osx/brew.sh...OK
     ✔ /Users/b4b4r07/.dotfiles/etc/init/osx/bundle.sh...OK
     ✔ /Users/b4b4r07/.dotfiles/etc/init/osx/go.sh...OK
     ✔ /Users/b4b4r07/.dotfiles/etc/init/osx/pygments.sh...OK
     ✔ /Users/b4b4r07/.dotfiles/etc/init/osx/unlocalize.sh...OK
     ✔ /Users/b4b4r07/.dotfiles/etc/init/linux/chsh.sh...OK
     ✔ /Users/b4b4r07/.dotfiles/etc/init/linux/goal.sh...OK
     ✔ /Users/b4b4r07/.dotfiles/etc/init/linux/pygments.sh...OK
 ➜ test brew.sh...
     ✔ check if init script exists...OK
     ✔ check running...OK
 ➜ test bundle.sh...
     ✔ check if init script exists...OK
     ✔ check if Brewfile exists...OK
Files=5, Tests=6
make: *** [test] Error 1
```

ローカルでは `make test` とすると，テストが走る．見かけないエラーが出てきたら `--silent` オプションを付けて実行する．ディレクトリの切り方は init のそれとほぼ同じで，Makefile からも，

```make
test:
	@DOTPATH=$(DOTPATH) bash $(DOTPATH)/etc/test/test.sh
```

としているだけである．init 以外のテストスクリプトは `test/` 直下に置く．例えば，raw ページのリダイレクトなどがある．

CI as a Service は Travis CI でテストしている．

```yaml
language: c
os:
    - linux
    - osx

env:
  global:
    - DOTPATH=~/.dotfiles
    - GOPATH=~

install:
    - curl -L dot.b4b4r07.com | bash
    - cd $DOTPATH
    - make init

script:
    - make --silent test
```

`bash -c ...` としていないのはシェルの再起動を防ぐためである．Travis CI でシェルを切り替えてしまうとテストするタスクも消え去るからである．

## vital.sh

`vital.sh` は最も重要なファイルの一つで，`../install` を参照するシンボリックリンクのライブラリファイルである．

挙動が面白いスクリプトで，コールのされかたによって `vital.sh` として動作したり `install` だったりする．`lib/install` として振る舞うのは `bash -c ...` か `... | bash` で呼ばれたときのみ．つまり，

- `bash -c "$(curl -L dot.b4b4r07.com)"`
- `curl -L dot.b4b4r07.com | bash`

のときである．`source` で取り込まれたときは `vital.sh` として振る舞い，それ以外（`bash vital.sh` など）では実行されない．HTTP を通してインストールする場合，前者 2 種のどちらかであるし，ライブラリとして読み込む場合は，`source` や `.` コマンドが必要で，コマンドラインから実行するようなユースケースは想定されないからだ．

`vital.sh` にはたくさんのユーティリティがあって，プラットフォームを検知する関数や `$PLATFORM` という環境変数，`has`，`die` など．

# まとめ

```console
$ bash -c "$(curl -L dot.b4b4r07.com)"
```

上で解説したテクニックや設計を集合させると，下の Gif アニメーションのようにあっさりと環境構築が可能になる．たったワンコマンドで数分，数秒で完了する．*initialize* までやりたいなら `-s init` をつけるだけ．

[![](https://raw.githubusercontent.com/b4b4r07/screenshots/master/dotfiles/demo.gif)][dotfiles]

これらの設計，つまり

- インストールスクリプト（ *installation script* ）を用意する
- HTTP 経由で利用できるように工夫する
- *deploy* と *initialize* は分ける

をベースに dotfiles をつくっていくと簡単に環境の再現ができるようになる．たかがターミナルの設定とはいえ，環境の再構築は簡単にできるに越したことはないと思う．ワンコマンドですぐに再構築できるのは環境が壊れることを恐れさせない強みになるからだ．

[dotfiles]: https://github.com/b4b4r07/dotfiles
