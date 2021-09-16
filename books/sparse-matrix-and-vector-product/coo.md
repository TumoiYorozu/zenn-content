---
title: 疎行列の格納形式：COO
---
# 概要
COO形式 (Coordinate形式，座標形式，ijv形式とも)は疎行列の格納形式の中で最も理解しやすく，簡単で，人間に優しいフォーマットです．
つまり計算機に厳しいフォーマットです．

計算機に厳しい理由は2つで，(1) 次章で述べるCRS形式と比べてデータ量が多いこと，(2) COO形式で格納された疎行列に対して疎行列ベクトル積を行う場合，スレッド並列化が困難なことです．

そのため行列の作成や，Matrix Market Formatなどの形式でのファイルへの読み書きに使われることが多いです．
これを使って最後まで計算を続けることは考えないほうが良いでしょう．これは[Scipy SparseのCOOのドキュメント ](https://docs.scipy.org/doc/scipy/reference/generated/scipy.sparse.coo_matrix.html#scipy.sparse.coo_matrix)にもそう書いてあります．

## COO formatの構成
COO形式は座標形式とも呼ばれるとおり，x座標，y座標，値の3つの値のペアで疎行列を格納する形式です．

以下にCOO形式のメモリレイアウトを示します．

![coo.png](https://raw.githubusercontent.com/t-hishinuma/zenn-content/main/books/sparse-matrix-and-vector-product/COO.png)

COO形式は3本の配列から成り，行数 $N$ , 非零要素数 $nnz$ の行列を以下の3つの配列で表現します．
- 長さ$nnz$ の行番号を示す整数型インデックス配列 $row\_index$
- 長さ$nnz$ の列番号を示す整数型インデックス配列 $col\_index$
- 長さ$nnz$ の値を示す配列 $val$

インデックス配列が32bit 整数, 値が64bit 浮動小数点数であれば，1要素を表現するのに必要なデータ量は $32 + 32 + 64 = 128 bit$ ,つまり行列全体では $16 Byte \times nnz$  となります．

COO形式は少し定義が曖昧で，ライブラリにもよるのですが，以下の2点に注意してください．
- 要素がソートされている保証がない
- 要素の重複がある場合がある (重複している場合の扱いは未定義)

ただ，Matrix MarketやSuite Sparse Matrix Collectionに掲載されているものは，ソート済かつ重複のないものしかない(と思う．特に記載がないが見たことがない)ので，本書のサンプルではソート済かつ重複のないもののみを取り扱います．

また，COOは特定の行や列に対するアクセスも苦手です．行や列を取得するためには $row\_index$ と $col\_index$ を全て判定しながら行列をサーチする必要があります．
ソートされている保証がない場合は行列をすべてサーチする必要があるので，非常にコストが高いと言えると思います．

# COO形式の行列の取り扱い
## Matrix Market fomratからCOO形式の行列の作成
Matrix Market FormatとCOO形式はほぼ同じなので，主にヘッダの処理が大変な作業になります．

[こちら](https://github.com/t-hishinuma/SpUtil)のリポジトリに，Matrix Market formatからCOOの3本の配列を作成するプログラムを起きました．
いわゆるヘッダライブラリになっており `SpUtil.h` をincludeするだけで使えます．好きに使ってください．

なお，本書のサンプルコードは，あえてCで，構造体なども使わない形にしておきました．
C++でクラスなんかを使ってきれいにやりたい人は著者が開発しているmonolishというライブラリの[このへん](https://github.com/ricosjp/monolish/blob/master/src/utils/IO/IO_coo.cpp)が参考になるかもしれません．

[IO_and_convert.c](https://github.com/t-hishinuma/SpUtil/blob/main/test/IO_and_convert.c)がサンプルコードに当たります．
このサンプルは2パートに分かれており，(1) MatrixMarket formatの `test.mtx` からCOOを作成し，(2) COOから次章でCRSに変換します．
本章では前半部分にあたる(1)のCOOを作成する部分について説明します．

```
int N, nnz;

//file open and read header to get matrix size
FILE* fp = fopen("./test.mtx", "r");
SpUtil_read_mm_header(fp, &N, &nnz);

printf("N = %d, nnz = %d\n", N, nnz);

// allocate COO array
int* coo_row_index = (int*)malloc(sizeof(int)*nnz);
int* coo_col_index = (int*)malloc(sizeof(int)*nnz);
double* coo_val = (double*)malloc(sizeof(double)*nnz);

// create COO from file
SpUtil_mm2coo(fp, N, nnz, coo_row_index, coo_col_index, coo_val);

// close
fclose(fp);
```

1. `SpUtil_read_mm_header()` でヘッダを読み，エラー処理と行列サイズ $N$ , 非零要素数 $nnz$ を取得し，
1. 取得したサイズを用いてユーザが配列をallocateし．
1. `SpUtil_mm2coo()` を用いて，データ部を読んでCOOの配列を作成します．

デバッグ用に，標準出力に値を吐き出す `SpUtil_print_coo()` という関数もあるので，よかったら使ってください．

## COO形式の疎行列ベクトル積 (逐次)
インデックス配列を用いて順番に計算していくだけです．

ただし，あらかじめ $y$ が初期化されていることは保証できないので，計算する前に $y$ を初期化してやる必要があります．
密行列などでは二重ループになるので行のループの最初に初期化するなり，一時変数を用いるなりすれば良いのですが，COO形式では行の境目を判定する方法がないので初期化を別途行うしかありません．

```cpp
for(int i = 0; i < N; i++){
    y[i] = 0.0;
}

for(int i=0; i<nnz; i++){
    y[ col_index[i] ] += A.val[i] * x[ row_index[i] ];
}
```

計算量としては疎行列ベクトル積だけを見れば $2nnz$ で，ベクトル $y$ の初期化を含めれば $2nnz+N$ になります．

## COO形式の疎行列ベクトル積 (スレッド並列)
COO形式の疎行列ベクトル積を単純にスレッド並列しようと思ったとき，以下のようなコードを思いつくと思います．

```cpp
#pragma omp parallel for
for(int i = 0; i < N; i++){
    y[i] = 0.0;
}

#pragma omp parallel for
for(int i = 0; i < nnz; i++){
    y[ col_index[i] ] += A.val[i] * x[ row_index[i] ];
}
```

残念ながらこのコードは間違いです．$y$ の初期化については問題なく並列化できますが．疎行列ベクトル積に関してはこれで答えが合う保障はありません．

試しに以下の3x3の密行列を考えます (ZennさんKatexのpmatrix使えんかった．．．)．

```
1 & 2 & 3\\
4 & 5 & 6\\
7 & 8 & 9\\
```

これをCOO形式にした場合， $val$ 配列は[1,2,3,4,5,6,7,8,9]になります．これを2スレッドで均等に分割した場合，それぞれの計算領域は
- 1スレッド目: [1,2,3,4,5]
- 2スレッド目: [6,7,8,9]

となります．この場合，運が悪いと2行目の計算の結果(1スレッド目の4,5と2スレッド目の6)を $y$ に同時に`+=`で書き込むことになり，書き込みが競合します．
これはタイミングの問題で，1スレッド目が1行目を計算している間に2スレッド目が2行目の計算を終わらせてくれれば競合が起きることはないのですが，これは保証されないため，汎用的なコードでスレッド並列するのは間違いです．

これを解決しようとした場合，スレッドに対して行をまたがないように計算領域を割り当てることが必要になりますが，$row\_index$ と $col\_index$ から行の切れ目を判定することは難しく，事前にデータを分割しておくくらいしか方法はありません．

そのためCOO形式は並列計算機に向いていない格納形式だと言われています．

# 余談，それでもCOO形式を使いたい場合
COOの問題点は行単位で計算することができず，単純な実装では並列化時にスレッドセーフにならないことです．
それでもCOO形式を並列化したい場合は，あらかじめ行列を行を跨がないようにブロック行分割しておくことになります．

流体解析でよく使われる [OpenFOAM](https://www.openfoam.com/) は，格納形式にCOOを採用しています．OpenFOAMの並列化は主にプロセス並列で，行列を予めプロセスごとに分割して保持するためCOOでも並列化できています．

ただし，GPUなどの数千コアのプロセッサだと，行列を数千に分割し，数千プロセスを立ち上げねばなならず現実的ではありません．
やはり現代においては理由がない限りCOO形式は計算に用いるべきではないでしょう．
