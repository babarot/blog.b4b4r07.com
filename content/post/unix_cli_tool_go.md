+++
date = "2015-06-23T15:47:51+09:00"
tags = ["golang", "cli"]
title = "開発者から見た UNIX 哲学とコマンドラインツールと Go言語"

+++

CLI（Command-line Interface）ツールが好きで自分でもよく作るし，よく使っている．最近は高速で，かつクロスコンパイルが容易な Go 言語がその開発に使われることが多いようだ．実際に筆者も拙劣ながら Go 言語で何個かリリースしている．

- [b4b4r07/gch](https://github.com/b4b4r07/gch) [![](https://img.shields.io/github/stars/b4b4r07/gch.svg?style=flat-square)](https://github.com/b4b4r07/gch "b4b4r07/gch - GitHub") 
- [b4b4r07/gotcha](https://github.com/b4b4r07/gotcha) [![](https://img.shields.io/github/stars/b4b4r07/gotcha.svg?style=flat-square)](https://github.com/b4b4r07/goal "b4b4r07/goal - GitHub") 
-  [b4b4r07/gomi](https://github.com/b4b4r07/gomi) [![](https://img.shields.io/github/stars/b4b4r07/gomi.svg?style=flat-square)](https://github.com/b4b4r07/gomi "b4b4r07/gomi - GitHub") 

CLI ツールの歴史はとても長く，過去たくさんの素晴らしい資産と独自の哲学がある．現代にいきる我々も当然その思想に従うべきで，CLI ツールを作るならその哲学を踏襲するのが常識だ．

# UNIX 哲学

1. *Small is beautiful.*
2. *Make each program do one thing well.*
3. *Build a prototype as soon as possible.*
4. *Choose portability over efficiency.*
5. *Store data in flat text files.*
6. *Use software leverage to your advantage.*
7. *Use shell scripts to increase leverage and portability.*
8. *Avoid captive user interfaces.*
9. *Make every program a Filter.*

これは UNIX 哲学の基本思想で Go 言語ツールを作る上でいいように解釈すると，以下のことだと思っている．

1. Minimal で Enough なソフトウェア設計であること
2. 他のプログラムと協調できるように，標準入出力を活用せよ
3. 外部データを使うならフラットテキスト（JSON、YAML、TOML など）を利用せよ
4. ポータビリティを優先し，苦痛なくインストールできることを目指せ
5. 対話インタフェースは避け，フィルタとして振る舞えること

これについて言及している記事があるのでこれらも非常に参考になる．また，[@deeeet](http://deeeet.com) (a.k.a. Taichi Nakashima) さんは CLI デベロッパとしてとても有益な記事・ツールが多いのでマスト watch だ．

- [コマンドラインツールについて語るときに僕の語ること](http://deeeet.com/talking/2014/08/31/yapc-2014/)
- [コマンドラインツールを作るときに参考にしている資料](http://deeeet.com/writing/2014/08/27/cli-reference/)

# コマンドラインツールをつくるときのガイドライン

Go 言語で作る場合も，ご多分に漏れずこの UNIX 哲学をベースに作っていくことになる．しかし，これは Go 言語だけの話ではないが，コマンドラインツールを作る上で At least であり Most important な概念がある．

それは，**標準入出力をきちんと使い分ける**ことだ．これに関して，パイプの開発者 M.D.マキルロイも次のように要約している．

>これがUNIXの哲学である。
一つのことを行い、またそれをうまくやるプログラムを書け。
協調して動くプログラムを書け。
標準入出力（テキスト・ストリーム）を扱うプログラムを書け。標準入出力は普遍的インターフェースなのだ。
>
>— M. D. マキルロイ、UNIXの四半世紀

コマンドラインでは，正常終了をゼロ値，異常終了を非ゼロ値として扱う．

```console
$ ./not-found-command
$ echo $?
1
```

エラーかどうかを判断するのにこの終了ステータスを使うからだ．

```console
$ ./not-found-command || echo "Error!"
Error!
```

そして，このときのエラーレポートは標準エラー出力に出力するべきだ．以下では，標準出力であるファイルディスクリプタ1番を標準エラー出力である2番に出力先を変更している．

```console
$ ./not-found-command || echo "Error!" 1>&2
Error!
```

こうすることで，簡単に出力先をコントロールでき，処理を切り分けることができる．

- [シェルスクリプトを書くときに気をつける9箇条](http://qiita.com/b4b4r07/items/9ea50f9ff94973c99ebe)
- [シェルスクリプトのオプション設計ガイドライン](http://qiita.com/mollifier/items/95a294f95f5977b9d663)

# Go 言語で作る

標準入出力，特に標準出力と標準エラー出力をきちんと使い分けることは，テスタブルなコマンドラインツールを設計することに直結する．しかし意外とこれが難しく，筆者もどう書けばいいか苦労していた．シェルスクリプトで実装する場合はとても簡単だ．

```bash
#...

if [ ! -f "$file" ]; then
	echo "$file: not found" 1>&2
	exit 1
fi

main
```

上の例では，特定のファイルが存在しない場合，簡単なエラーレポートを標準エラー出力に流し，ステータスコード `1` で `exit` している．ちなみに，コマンドラインツールを設計する上で，エラー時の出力は最小限であるべき（静かなエラー）で，正常終了時は出力をしないべきなのである．これは他のツールと協調するため（出力を無視させたり整形させる手間をとらせない）とされている．

これを Go 言語化する上で難しいのは次の点だ．

- 終了ステータスをゼロと非ゼロで分ける
- そのときのエラーレポートは標準エラー出力に行う

Go 言語では関数の戻り値を複数取れるが，エラーをともなう処理を行う関数の場合，二値目には Error を返すことが一般的だ．

```go
func someFunc() (string, error) {
	//...
	if err != nil {
		return str, err
	}
	//...
}
```

そしてそれを最終的に一番ユーザに近いレイヤーである `main()` で受け取り，エラーレポートとともに，`exit` する．

```go
func main() {
	//...
	_, err := someFunc(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
	//...
}
```

結果としては期待されていた実装ができているように思える．しかし，これはアンテスタブルで，「コマンドラインツールのエラー」としては扱いきれていない．筆者も悩んでいたところ，これもまた [@deeeet](http://deeeet.com) さんのエントリではあるが，その問題点を完璧に克服していたので紹介したい．

# Go 言語で設計する

- [Go言語でテストしやすいコマンドラインツールをつくる](http://deeeet.com/writing/2014/12/18/golang-cli-test/)

この記事では**テスタブルな**ツールづくりにフォーカスしている．また，ファイル構成は最低限2つのファイル，`main.go` と終了ステータスと標準入出力をハンドリングした `cli.go` から成る．

## 終了ステータスをハンドリングする

まず，問題なのが，終了ステータス値（*zero or non zero*）を Go 言語側から扱えていない点だ．これについて，`int` 型の終了値を返す関数（後述するがメソッドとして）を定義する．

```go
func (cli *CLI) Run(args []string) int {
	//...
	if _, err := os.Stat(someFile); err != nil {
		err := fmt.Sprintf("%s: no such file or directory", someFile)
		fmt.Fprintln(cli.errStream, err)
		return ExitCodeFileNotFound
	}
	//...
```

`os.Exit(1)` するのではなく，`return err` するのでもなく，`int` でエラーの数値を返すのだ．もちろん，`0` 以外の数値であるが，これについても可読性のために以下のように定数として定義しておく．

```go
const (
	ExitCodeOK    int = 0
	ExitCodeError int = 1 + iota
	ExitCodeFileNotFound
	ExitCodeParseError
)
```

こうすることで，終了値をエラーハンドリングしやすくなるし，コードとしての見通しも良くなる．エラーレポートに関しては，そのままその関数内で `fmt.Fprintf()` して問題ない．端末画面上に出力するだけのステイトメントについては，この場合ハンドリングするべき対象でもないし，そのまま `main()` に伝わり，`cli.errStream` に出力される．

## ディスクリプタをハンドリングする

次は標準入出力の制御だ．`os` パッケージにもあるとおり，`os.Stdout` と `os.Stderr` によって操作できるが，テスタブルに書くためには以下のようにするのがよい．

まず，`io.Writer` の `outStream` と `errStream` をフィールドとしてもつ構造体 `CLI` を定義し，その `CLI` 構造体をレシーバとし，またコマンドライン引数をその引数としてもつ `Run()` メソッドを定義する．

```go
type CLI struct {
	outStream, errStream io.Writer
}

func (cli *CLI) Run(args []string) int {
	//...
}
```

前項でも書いたとおり `Run()` では（オプション引数がある場合はそのパース処理と）具体的なコマンドの処理内容を記述し，戻り値としてステータスコードを返すようにする．

あとは，`main()` 内での記述だが，大方の処理や操作を `Run()` メソッドで奪ったのでこれだけでよい．

```go
//...

func main() {
    cli := &CLI{
    	outStream: os.Stdout,
    	errStream: os.Stderr,
    }
    os.Exit(cli.Run(os.Args[1:]))
}
```

## コマンドラインツールをテストする

コマンドラインツールにおいてもテストは大事だが本稿の趣旨から外れるのと，先に挙げた記事と丸かぶりしかねなくなるので，それについては省略する．

- [I/O を伴うテストには bytes.Buffer が便利](http://qiita.com/yuya_takeyama/items/c4211fa77488cb6915ec)
- [Unit-testing programs depend on I/O in Go](https://yuya-takeyama.github.io/presentations/2014/11/30/gocon_2014_autumn/)
- [Go言語でテストしやすいコマンドラインツールをつくる](http://deeeet.com/writing/2014/12/18/golang-cli-test/)

# 優れたコマンドラインツールとは

UNIX 哲学に準拠していて，ストリームをきちんと理解できているツールは素晴らしい．筆者自身も試行錯誤しながら作っている．これは実装言語に依らない思想や考え方なので，CLI デベロッパは「UNIX という考え方」を参照するべきだ．

![](http://ecx.images-amazon.com/images/I/518ME653H3L.jpg)

## 参考

- [コマンドラインツールを作るときに考えているちょっとした設計方針](http://www.songmu.jp/riji/entry/2015-04-18-commandline-tool-design.html)
- [コマンドラインプログラムにおける引数、オプションなどの標準仕様](http://yohshiy.blog.fc2.com/blog-entry-260.html)
- [Go言語によるCLIツール開発とUNIX哲学について](http://yuuki.hatenablog.com/entry/go-cli-unix)
