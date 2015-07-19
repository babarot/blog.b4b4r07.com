+++
date = "2015-06-17T23:10:42+09:00"
tags = ["vim", "vim plugin"]
title = "Vim プラグイン開発の Tips"

+++

プラグイン開発が盛んなエディタの一つに Vim がある．

筆者も拙劣ながら2つほどリリースした．

- [b4b4r07/vim-shellutils](https://github.com/b4b4r07/vim-shellutils)
- [b4b4r07/vim-autocdls](https://github.com/b4b4r07/vim-autocdls)

前者はシェルコマンドを Vim script で関数定義してエミュレートしたものだ．Vim に最適化されているため，`system()` や `!cmd` するよりも便利になっている．後者は，それを応用したもので `:cd` するたびに `:Ls` するようにしたプラグインだ．

さて，宣伝はここまでにして，Vim プラグインを作成するとき，普段と同じ環境（つまり同じ `.vimrc` を使用し，普段通りのプラグインを読み込んでいる状態）で作っていると何かと弊害が起こりがちである．Vim の動作も安定しない．またプラグイン作成を一時中断して，他のタスクを処理するために Vim を使うとなると，開発中のプラグインが作業に干渉したりして不便な状態を強いられることが多い．

ここらへんをどう処理しているか，アマチュア Vim プラグイン開発者にとって凄く気になるところだけれど，意外とノウハウは公開されていない（おそらく有名所だと @thinca さんくらい）．

- [Vim プラグインの開発スタイルのお話](http://d.hatena.ne.jp/thinca/20100216/1266294717)

# Vim プラグインの開発スタイル

筆者の場合，手探りながらも個人的には満足できるスタイルが確立できたので紹介することにした．

開発スタイルの落とし所としてはこうだ．

- `.vimrc` は普段通りのものを使いたい
	- 開発するときに読み込む `.vimrc` を作るのは面倒だし，それを読み込ませた Vim を起動するのはもっと面倒だ
	- それに普段の使い勝手（コマンドやキーバインド）を失いたくない
- 開発中は余計な操作を強いられずに，`.vimrc` を制限したい
	- 作成中のプラグインの機能を逐一確かめるために，既存 Vim プラグインは読み込まない
	- その制限をシームレスに息を吸うかのごとく設けたい

そこで，`.vimrc` の冒頭に

```vim
let s:true  = 1
let s:false = 0

let s:vimrc_plugin_on = get(g:, 'vimrc_plugin_on', s:true)

if len(findfile(".development.vim", ".;")) > 0
  let s:vimrc_plugin_on = s:false
  set runtimepath&
  execute 'set runtimepath+=' getcwd()
  for s:plugin in split(glob(getcwd() . "/*"), '\n')
    execute 'set runtimepath+=' . s:plugin
  endfor
endif
```

を記述する．

やっていることは至って単純で，`.development.vim` が置かれたディレクトリ配下はすべて `s:vimrc_plugin_on` を `false` に制限する．もちろんデフォルトは `true` だ．

そして，`.vimrc` にあるプラグインの設定を記述している箇所に，

```vim
 if has('vim_starting') && isdirectory($NEOBUNDLEPATH)
+  if s:vimrc_plugin_on == s:true
     set runtimepath+=$NEOBUNDLEPATH
+  endif
 endif
```

上記の `if` 文を加える．これによって読み込むプラグインの置かれたディレクトリを操作している．

たったこれだけで Vim の様相は大きく変わる．`g:vimrc_plugin_on` が `true` だと，今までどおりの Vim だけれど，`false` になると `.development.vim` が置かれたディレクトリにあるディレクトリしかプラグインと認識されないからだ．

- `g:vimrc_plugin_on == s:true`

![](/images/vim_plugin_on/true.png)

- `g:vimrc_plugin_on == s:false`

![](/images/vim_plugin_on/false.png)

画像のとおり `true` のときは @thinca さん作の [vim-splash](https://github.com/thinca/vim-splash) が読み込まれ，AA が表示されている．一方で，`false` のとき，つまり `.development.vim` が存在するとき（プラグイン開発時）にはそれが読み込まれていないことがわかる．

この方法を取ることで，Vim プラグイン開発中は既存のプラグインの読み込みを制限できる．開発はじめに開発ディレクトリに `.development.vim` を作成するだけでいい．`touch` コマンドでできる．開発が終われば削除するだけでいい．

もし，既存の Vim プラグインを使用したい場合，正規のプラグインディレクトリから開発ディレクトリまでシンボリックリンクをはればよい．そのプラグインは読み込まれるので，使用することができる．

この開発スタイルは正しいのかは分からない．ただ，こうすることで自分のケースでは何の不便もなく開発に打ち込むことができる．開発を一旦中断する場合，つまり他のディレクトリに移ることで，いつも通りの Vim に戻る．開発中も既存のプラグインさえ使えないものの，他の設定はフルに生きる．

これ以外のおすすめの手法があればぜひ．
