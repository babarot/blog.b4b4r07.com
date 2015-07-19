+++
date = "2015-05-22T00:00:00+09:00"
tags = ["golang", "cli"]
title = "Golang でコマンドラインにゴミ箱を実装した話"
+++

# まえがき

デスクトップに一際目立つアイコンで鎮座する，[ゴミ箱][gui]は使っているだろうか．今となってはゴミ箱は GUI デスクトップの象徴的存在だ．誤削除を防ぐ手段としても，安心した削除支援の存在としても GUI デスクトップに無くてはならない．

[gui]: http://ja.wikipedia.org/wiki/ごみ箱_(GUI)

さて，GUI デスクトップに相当する CLI はホームディレクトリだが，これにゴミ箱がないのは不便ではないだろうか．`rm` に関してはその概念をなくして削除を行い，他に「ゴミを捨てる」にあたるようなコマンドはない．

>GUI以前のコマンドラインには、ゴミ箱という考え方はなかった。（と思う）ファイルやフォルダを削除するにはrmコマンドを使っていた。そのまま使えば、rmを実行した瞬間にファイルは削除される。あるいは、-iオプションによって、削除する前に確認メッセージも表示できるが、yを選択した瞬間にファイルは削除される。
>
>だから、ゴミ箱というフォルダに移動して一時的に保存しておくという考え方は、画期的なことだと思った。なぜなら、間違って必要なファイルを捨ててしまうこともあるからだ。rmコマンドでは削除を取り消せないが、ゴミ箱に移動しただけなら、必要なファイルを元の場所に戻せば救われるのである。この辺は、リアルな世界のゴミ箱の機能に近い。（ゴミ箱をあさった経験は、少なからず誰しもあると思う） via [後悔しない最高のゴミ箱環境を模索する](http://d.hatena.ne.jp/zariganitosh/20110106/best_trash)

むしろなんで今までなかったのだろう．CLI の習熟度に関わらず，まれに重要なファイルを誤って消しそうになることもあるし，他には例えばチルダ付きのファイルを消そう（`rm ~/*~`）として余計にスペースが入った（`rm ~/* ~`）と，いいようなタイポによる誤削除は往々にしてあり得る．

筆者にも同様の経験があり，それ以後ゴミ箱を実装したスクリプトを書いて利用していた．

- [b4b4r07/rmr.sh](https://github.com/b4b4r07/dotfiles/blob/master/etc/scripts/rmr.sh)

このスクリプトは，削除に関してはいいのだが（`~/.rmtrash` に *YYYY/MM/DD/file.H_M_S* でアーカイブする），取り出すときに元ファイルのパス情報その他が失われてしまうため，リストアが困難なのが欠点だった（今は少し改良をしていて `peco` 経由でリストアするようになっている）．

GUI のそれのように CLI でも削除後に簡単に「戻す」ができたら便利だと思い，調べてみたところ以下のリポジトリが GitHub でスターを多く獲得していた．

- [andreafrancia/trash-cli](https://github.com/andreafrancia/trash-cli) [![](https://img.shields.io/github/stars/andreafrancia/trash-cli.svg?style=flat-square)](https://github.com/andreafrancia/trash-cli "andreafrancia/trash-cli") 

> Command line interface to the freedesktop.org trashcan.

- [sindresorhus/trash](https://github.com/sindresorhus/trash) [![](https://img.shields.io/github/stars/sindresorhus/trash.svg?style=flat-square)](https://github.com/sindresorhus/trash "sindresorhus/trash")

> Cross-platform command-line app for moving files and directories to the trash - A safer alternative to `rm`


# trash-cli

前者は Python 製のゴミ箱スクリプトのようだ．

- `trash-put`
- `trash-empty`
- `trash-list`
- `trash-restore`
- `trash-rm`

詳しいインストール方法は README を参照していただきたいが、`easy_install trash-cli` とした後に、5 つの専用コマンドがインストールされ，それぞれを削除やリストアのときに使い分けるようになっている．古典的な実装・Usage と言わざるを得ない感じで，正直 5 つのコマンドを使い分けるのは面倒です。

加えて，肝心なリストア部分は，

```console
$ trash-restore
0 2007-08-30 12:36:00 /home/andrea/foo
1 2007-08-30 12:39:41 /home/andrea/bar
2 2007-08-30 12:39:41 /home/andrea/bar2
3 2007-08-30 12:39:41 /home/andrea/foo2
4 2007-08-30 12:39:41 /home/andrea/foo
What file to restore [0..4]: 4
$ ls foo
foo
```

といった具合に，復元したいファイルをナンバーで指定して選択できるようになっている．このユーザインターフェイスは「戻す」ことを簡単に行える設計と言える．UNIX の哲学ではインタラクティブなコマンド設計は良しとされないが，今回は対話性が必要なユースケースの一つであろう．

基本的に Python の実行環境を整えればどこでも動作するが，きちんと動く Python 環境をセットアップするのは（少なくとも筆者は）面倒にしか感じない．

そしてもうひとつ欠点は開発が停滞している点だ．メンテナンスもされていない．

# trash

後者は Node.js によって実現したゴミ箱スクリプトだ．

OS 標準のゴミ箱に捨てれる点が良い．しかし，リストアができないようだ（OS のゴミ箱の「戻す」機能を使えば可能となるが，GUI 操作を要する）．また，ただゴミ箱に投げるだけなのでファイル名のコンフリクトも発生する．

```console
$ trash --help

  Usage
    trash [--force] <path> [<path> ...]

  Example
    trash unicorn.png rainbow.png
```

システム連携に目を輝かせたが，実際の使い勝手はよろしくなかった．

# そこで他の案

他にはどんな方法があるのか調べてた．

- [rcmdnk/trash](https://github.com/rcmdnk/trash)
- [dankogai/osx-mv2trash](https://github.com/dankogai/osx-mv2trash)
- [safe-rm](https://launchpad.net/safe-rm)

個々人で簡易的に実装したものだったりが多い．筆者のケースを解決するのに満足なツールはなかった．

そこで，全ての欠点を解消し，尚かつ利点を包括的に取り込んだ，オリジナルのゴミ箱スクリプトを作るしかないと考えた（Unix らしく）．

# gomi

[![](https://raw.githubusercontent.com/b4b4r07/gomi/master/images/gomi_logo.png)](https://github.com/b4b4r07/gomi "b4b4r07/gomi")

- [b4b4r07/gomi](https://github.com/b4b4r07/gomi) [![](https://img.shields.io/github/stars/b4b4r07/gomi.svg?style=flat-square)](https://github.com/b4b4r07/gomi "b4b4r07/gomi")

> gomi is a simple trash tool that works on CLI, written in golang

コイツはもの凄く便利だ．構想段階ではここまで便利になるとは思わなかった．

- クロスプラットフォーム，Mac でも Windows でも Linux でも
- シングルバイナリ，バイナリひとつあれば Python も Node.js も要らない
- リストアが簡単，[peco/peco](https://github.com/peco/peco) みたいなセレクタを備える
- クイックビュー（OS X でいう QuickLook ができる），リストアする前に捨てたファイルの中身を確認
- リストアしたい複数のファイルを選択して，カレントディレクトリに引っ張ってくるとかできる
- OS 標準のゴミ箱と連携できる
- もちろん，リストアは gomi からでも，GUI の「戻す」からでも OK
- YAML 形式の設定ファイルでカスタマイズできる
	- 例：リストア時に参照するログファイルに追加したくないファイルなど（例えば，.DS_Store といった復元を求めないファイル）
	- 例：ゴミ箱に入れるファイルのサイズ上限（デフォルトは1GB）

これだけでもかなり便利です．下の GIF を参照されたい．

[![b4b4r07/gomi](https://raw.githubusercontent.com/b4b4r07/gomi/master/images/gomi.gif)](https://github.com/b4b4r07/gomi "b4b4r07/gomi")

めっちゃ便利．

## インストール

Homebrew の Mac ユーザなら，

```console
$ brew tap b4b4r07/gomi
$ brew install gomi
```

で OK ．

Go 言語ユーザや開発者向けには，

```console
$ go get -u github.com/b4b4r07/gomi
```

Go でインストールできる．

その他のユーザに関しては，各プラットフォーム向けにコンパイルした実行ファイルが GitHub の [releases page](https://github.com/b4b4r07/gomi/releases) にアップロードしてあるので，バイナリをダウンロードして `$PATH` のどこかに移動すればよい．

## 使い方

```console
$ gomi files    # 削除する（複数ファイルまとめて削除もOK）
$ gomi -r       # 元ある場所に戻す
$ gomi -r .     # カレントディレクトリに戻す
$ gomi -s file  # OS のゴミ箱に捨てる
```

詳しくは，[README.md](https://github.com/b4b4r07/gomi) まで．

# 最後に

[Go 言語で rm 用ごみ箱ツール gomi を作った](http://qiita.com/b4b4r07/items/3a790fe7e925b4ba14f3)

一度，Qiita に公開した．この時はまだ β 版だったが，安定版を提供できるところまではきたかなといった感じなので．
