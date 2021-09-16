---
title: 疎行列の格納形式：CRS
---

# 概要
CRS形式 (Compressed Row Storage形式，行優先圧縮格納形式，CSR形式とも)は，COO形式の行の境界がわからない(並列化できない)問題を解決しつつ，COO形式よりデータサイズが小さいフォーマットです．
つまり人間にはちょっとだけ厳しいフォーマットです．

基本的にCRSを扱うための共通のファイルフォーマットなどはありません．そのためMatrix Market Formatから直接作成するか，一度COO形式を作ってからCRS形式に変換することで生み出されます．

こちらもCOOと同じく，[Scipy SparseのCSRのドキュメント ](https://docs.scipy.org/doc/scipy/reference/generated/scipy.sparse.csr_matrix.html)を参考までに貼っておきます．

## CRS formatの構成
CRS形式は3つ配列で疎行列を格納する形式です．
値を示す配列と列番号を示す配列はCOO形式と同じですが，行番号を示す配列をもたず，代わりに各行の非ゼロ要素の開始位置を示す配列をもちます．

以下にCRS形式のメモリレイアウトを示します．

![CRS.png](https://raw.githubusercontent.com/t-hishinuma/zenn-content/main/books/sparse-matrix-and-vector-product/CRS.png)

CRS形式は3本の配列から成り，行数 $N$ , 非零要素数 $nnz$ の行列を以下の3つの配列で表現します．
- 長さ$N+1$ の各行の非零要素の開始位置を示す整数型インデックス配列 $row\_ptr$ ，ただし，$N+1$ 番目は非零要素数を示す．
- 長さ$nnz$ の列番号を示す整数型インデックス配列 $col\_index$
- 長さ$nnz$ の値を示す配列 $val$

各行の開始位置を示す配列をもつ必要があることから，行に関してはソートされていることが保証されています．
行内ではソートされていなくても $col\_index$ に従うだけなので大丈夫ですが，殆どの場合ソートされていると思います．保証されているかは分かりません．

メモリデータ量は32 bit整数と64 bit浮動小数点の場合は $8nnz + 4nnz + 4(N+1)$ となります．

CRS形式の弱点は要素の追加や削除です． どこか途中に値を1つ挿入するとき，挿入する箇所以降の行の $row\_ptr$ を全てインクリメントする必要があります．
また，COOと同じく列に対するアクセスも苦手で，1列を取得するためには $col\_index$ を全て判定しながら行列全体をサーチする必要があります．

# CRS形式の行列の取り扱い
## COO形式からCRS形式の行列の作成
ここでは，COO形式からCRS形式を作る方法について述べます．
Matrix Market Formatから直接CRSを作る方法は．．．いつか書きます．

[こちら](https://github.com/t-hishinuma/SpUtil)のリポジトリに，COO形式からCRS形式を作成するプログラムを置きました．

[IO_and_convert.c](https://github.com/t-hishinuma/SpUtil/blob/main/test/IO_and_convert.c)がサンプルコードに当たります．

基本的には配列を確保して，COOとCRSの配列を構成する3+3本の配列を渡すだけのものになっています．

```cpp
    // allocate CRS array
    int* crs_row_ptr = (int*)malloc(sizeof(int)*(N+1));
    int* crs_col_ind = (int*)malloc(sizeof(int)*nnz);
    double* crs_val = (double*)malloc(sizeof(double)*nnz);
    
    // crate CRS from COO
    SpUtil_coo2crs(N,nnz,
                coo_row_index, coo_col_index, coo_val,
                crs_row_ptr, crs_col_ind, crs_val);
```

## CRS形式の疎行列ベクトル積 (逐次)

CRS形式は $row\_ptr$ 配列の`i`番目と`i+1`番目を見れば，`i`行目に存在する非零要素の数がわかるというのが最大のキモです．

もう一度，先程の行列を例に説明します．
![CRS.png](https://raw.githubusercontent.com/t-hishinuma/zenn-content/main/books/sparse-matrix-and-vector-product/CRS.png)


インデックス配列を用いて順番に計算してくだけです．
ただし，あらかじめ $y$ が初期化されていることは保証できないので，計算する前に $y$ を初期化してやる必要があります．
密行列などでは二重ループになるので行のループの最初に初期化するなり，一時変数を用いるなりすれば良いのですが，COO形式では行の境目を判定する方法がないので初期化を別途行うしかありません．

```cpp
for(int i=0; i<N; i++){
    y[i] = 0;
    for(int j = row_ptr[i]; j < row_ptr[i+1]; j++){
        y[i] = val[j] * x[col_index[j]];
    }
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

試しに以下の3x3の密行列を考えます(ZennさんKatexのpmatrix使えんかった．．．)．

\begin{pmatrix}
1 & 2 & 3\\
4 & 5 & 6\\
7 & 8 & 9\\
\end{pmatrix}

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

## hoge

![crs.png](https://raw.githubusercontent.com/t-hishinuma/zenn-content/main/books/sparse-matrix-and-vector-product/CRS.png)

## 出典
CRSをrefしたいときはこれを使うとよい：

```
Richard Barrett, Michael W. Berry, Tony F. Chan, James Demmel, June Donato, Jack Dongarra, Victor Eijkhout, Roldan Pozo, Charles Romine and Henk A. van der Vorst, Templates for the Solution of Linear Systems: Building Blocks for Iterative Methods, SIAM: Philadelphia, PA, 1993.
```
