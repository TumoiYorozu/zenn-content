---
title: "数値計算ライブラリにおいて配列を表現するクラスの実装とはどうあるべきかを悩んだ末のポエム (1次元)"
type: "tech"
topics: ["Tech", "C++"]
published: true
---

# はじめに
**この記事の答えは私自身見つかっていません．是非コメント欄にコメントください．．**

数値計算や機械学習のコードを書いている皆さん，どうもこんにちは．
今日は私が3日悩んでいるポエムのような話をしようと思います．

この記事では前提としてC++で開発することを考えますが，RustとかJuliaでも同じだと思います．

また，ちょっとわかりにくくて申し訳ないのですが，この記事では配列とarrayという言葉を分けて，
- array(アレイ) : ベクトルとの対比に用いる，掛けたりするとアダマール積になるもの 
- 配列 : メモリに連続に格納された領域のことを指す．ベクトルなのかテンソルなのかアレイなのかは問わない

と定義することにします．

プロトタイプ実装もせず，頭の中でしか考えてないです．

# なぜarrayクラスを再実装しないといけないのか
## 理由
さっそくですが，言語はともかくとして数値計算ライブラリにおいて，

```
x = {1, 2, 3};
```

と書いてあったとき，この `x` とはなんでしょうか？

一次元の配列であることは間違いないですが，実際には，
- 配列 (array)
- ベクトル (vector)
- テンソル (tensor)

という可能性があります．

人によってどれを望んでいるかは異なるし，人類はこの問題に対する答えを残念ながら定義できていません．これらの違いによって，たとえば `*` という演算が内積なのかアダマール積(要素ごとの積)なのかが異なります．

どのように定義するかはさておき，こういう問題がある以上，数値計算ライブラリの開発者は自分で何らかのクラスを実装する必要が出てきます．

## この記事で扱う適当な配列クラス
本記事では，このようなクラスをC++実装するときにどのような実装にすべきなのかを考えたいと思います．

C++であれば，`std::vector`, `std::array`, `std::valarray` などがありますが，それらをそのまま使えばいい例を出しても面白くないので，
ここでは一例として疎な配列のクラスを定義したいと思います．

疎な配列とは，例えば以下のような値がほとんどゼロの配列のことです．
```
{0, 0, 1, 0, 2, 0, 0, 3}
```

こういった配列の場合，ゼロの要素を保持しておく意味がないので，以下のようにゼロ以外のみで構成された配列とインデックスの配列だけ持てばよいです．
```
value = {1, 2, 3}
index = {2, 4, 7}
```

このようなクラスを自分で定義するときの実装について考えます．

# 実装として考えられるもの 
当然ですが，こういう関数を1から実装するのは大変です．
先に述べたようにC++にも `std::vector`, `std::array`, `std::valarray` などがあり，これらをそのまま使えれば標準ライブラリの沢山の機能が使えます．

一方でそのまま使えないから自分で作っているわけで，なんとか拡張しないといけません．
5つほど考えたので，以下に書いていきますが，1.と4.のどちらかな気がしています．

どちらが正解なのか，どういうメリット・デメリットがあるのか，深く考えたことがある人，コメントください．．．

## 1. 我が道を行き，自分で全部書く
ポインタなりスマートポインタをメンバ変数として持って，全部自分で書きます．
力こそパワーです．何の問題もありません．自分で書くのだから書いた通りの性能が出ます．

xtensorとかEigenとかはこれで実装している気がします (全部見きれてない)．

開発が滞ると陳腐化していきます．工数を無視してこれを管理し続けられるなら苦労しません

## 2. 自分で全部書くが，std::*へのキャストオペレータなりconvert関数を定義する
上手くディープコピーが発生しないようにカスタムアロケータを定義して中身だけぐるぐる使いまわします．

やめろ

## 3. publicなメンバ変数としてstd::*のクラスをもつ 
中身はstd::vector等だから抜き出すなり機能を使うなり勝手にしてねってことです．

今回の例だと，valueだけresizeされたりすると死にます．

やめろ

## 4. privateなメンバ変数としてstd::*のクラスをもつ
ユーザに見せたい関数だけラッパー関数を定義して見せます．それなりに工数はかかりますが，1.ほどじゃないはずです．
ただ，普通に実装するとCポインタからstd::vectorへのディープコピーが発生するのでシャローコピーをするカスタムアロケータが必須でしょうか？

3.もそうですが，gccやclangの実装の違いに苦しんだりしないかなあ．

これが1.とくらべてどういう問題が起きそうなのか想像力が追いついていません．

あとstd::*頼みな部分が強くなるので，FFIが大変だったりする？
結局クラスにした時点でCに戻さないといけないんだから手間は一緒か？

## 5. std::*のクラスを継承する
デストラクタとアップ/ダウンキャスト問題と戦うことになります (protected継承すればいい気はする)．
一見便利に見えますが，結局意味の違う関数は直さないといけないので4.とくらべて実装量が変わらない上，C++のバージョンアップに付き合う必要があります．

多分良くない気がするんですけど，そんなことない？