---
title: 疎行列のファイル形式：Matrix Market format
---

# 概要
Matrix Market Format (MM format, MM形式)は疎行列をファイルに書き出すときの一般的なフォーマットです．
他にもMatlab形式やHarwell Boeing形式と呼ばれるものがありますが，現在最も主流なのはこのMM形式．．．な気がします．

この形式は[scipyでも入出力の形式としてサポートされており](https://docs.scipy.org/doc/scipy/reference/generated/scipy.io.mmwrite.html)，Pythonなどからも簡単に利用することができます．

名前の通り [Marix Market](https://math.nist.gov/MatrixMarket/) というベンチマーク行列をまとめたWebページで提案されたフォーマットです．
Matrix Marketには十分なMM形式の説明や読み込みのためのソースコードなどが揃っていますが，すべて英語なので本ページでは一応日本語で簡単にまとめておきます．

この形式は一般行列以外にも対称行列やベクトルを表現することも出来るのですが，今回の記事では全ての非零要素を含む(対称の省略などをしない)，正方，実数の行列について扱います．

Matrix Market formatは，難しいことは特になく，基本的には行数，列数，値のペアを格納する，いわゆる座標形式となっています．
零要素こそ保持しませんが，テキストデータかつ座標形式なのでバイナリデータなどと比べるとファイルサイズが大きくなりやすいことに注意が必要です．

なお，Matrix Marketはかなり古いWebページのため，掲載されているベンチマーク行列のサイズが非常に小さく，昨今ではベンチマーク行列を得るという本来の目的には使われなくなっています．
現在でも頻繁に更新が続いている後継のWebページとして [Suite Sparse Matrix Collection](https://sparse.tamu.edu/) があります．ベンチマーク行列を探す場合はこちらを参照するのが良いでしょう．

## ファイル形式
原典は [こちら](https://math.nist.gov/MatrixMarket/formats.html) にあります．

Matrix Market形式はヘッダ部とデータ部に分かれています．

ヘッダ部では：
- `%%` から始まるヘッダ (1行目，バナーともよばれる)
- `%` から始まるコメント (ヘッダ2行目以降)

を記述し，データ部では，

- 行数と列数と非零要素数 (1行目)
- 1から始まる行番号，列番号，値 (2行目以降)

を記述します．

以下の，3x3の行列をMatix Market formatで表現することを考えます．

```
1 0 0
2 3 4
0 5 0
```

これは以下のように表現できます．

```sh
%%MatrixMarket matrix coordinate real general
% this is comment
% hello world
3 3 5
1 1 1.0
2 1 2.0
2 2 3.0
1 3 4.0
3 2 5.0
```

それぞれ詳しく見ていきます．

### ヘッダ (バナーとも)

```sh
%%MatrixMarket matrix coordinate real general
```

1ブロック目の `%%MatrixMarket` はMatrix Market形式であることを示すものです．

2ブロック目は表現するものを指します．`matrix` 以外が指定されているところはほとんど見ませんが，`vector` や `directed graph` が指定できることになっています．
2度も言いますが `matrix` 以外を読める実装を見たことはほとんどないです．

3ブロック目は座標形式で表現されている事を意味する `coordinate` です．これも `coordinate` 以外を見たことはないですが，ベクトルなどでは `array` にするそうです．
2度言いますが `coordinate` 以外を読める実装はry．

4ブロック目では値の型を指定します．ここには:
- real
- complex
- integer
- pattern

などが指定できます．`real`, `complex`, `integer`は説明不要かと思います．`pattern` が指定されている所は見たことがありません．

5ブロック目は行列の構造を指定します．
- general
- symmetric
- Hermitian
- skew-symmetric

`general` はすべての非零要素が格納されています．Matrix Market形式に対応しているソフトウェアであれば， `general` にはまず対応していると思っていいでしょう．

`symmetric` や `Hermitian`の場合，対角要素を含む下半分のみの非零要素を格納します．MatrixMarketやSuite Sparse Matrix collectionにも多くアップロードされていますが，対称行列に対する疎行列ベクトル積のライブラリはほとんどないため，読み込みも対応していないライブラリが多くあります．

`skew-symmetric` は交代行列ですが．．．実装を見たことはありません．

この本では，最も簡単な以下のヘッダの形式について取り扱います．

```sh
%%MatrixMarket matrix coordinate real general
```

### コメント

ヘッダ部では，2行目以降に `%` で始まる好きなコメントを書けます．

Matrix MarketやSuite Sparse Matrix collectionにアップロードされている行列を見る限り，Authorや行列の説明を書くことが多いようです．

### 行数と列数と非零要素数 (1行目)
データ部の1行目には，行列の行数，列数，非零要素数を書きます．

今回は3x3の行列で，非零要素数が5個なので，以下のように書きます．

```sh
3 3 5
```

### 1から始まる行番号，列番号，値 (2行目以降)
データ部の2行目以降には，実際の行列の値を座標形式で記述します．

行数，列数，値をそれぞれ記述し，今回の行列では以下のように書きます．

```sh
1 1 1.0
2 1 2.0
2 2 3.0
1 3 4.0
3 2 5.0
```

注意が必要なのは，あくまで座標としての表現なのでソートされている保証はないことです．
多くの場合ソートされていますが，稀にソートされていない実装もあります．この本で紹介するコードは，ソートされていることを前提に書いてあることに注意してください．
