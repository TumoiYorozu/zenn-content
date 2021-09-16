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

例として2行目を計算したいとします．
ここで2行目には3つの要素があります．これは $row\_index[3] - row\_index[2]$ をすることで，$5-2=3$ として求めることができます．
2行目の開始点は $row\_index[2]$ を参照すると2ですので，$val[2]$と$cod\_ind[2]$を見に行くと，2行目の最初の要素が`c`で，これが`0`列目だということがわかります．

このように順にやっていくと以下のような疎行列ベクトル積のプログラムが書けます．

```cpp
for(int i=0; i<N; i++){
    y[i] = 0;
    for(int j = row_ptr[i]; j < row_ptr[i+1]; j++){
        y[i] += val[j] * x[col_index[j]];
    }
}
```

これのキモは，$row\_index$ を参照する行方向の`i`ループと，行内の要素を計算する`j`ループに分かれていることです．
「行ごと」に処理が行えるので，$y$ の初期化を2行目で行えばよく，COOのようにyの初期化を別のループとして実行しなくてよいです．

計算量としては $2nnz$ になります．

## CRS形式の疎行列ベクトル積 (スレッド並列)
CRS形式の疎行列ベクトル積は簡単です．行のループと列のループが別れているので，行に対して並列化すればいいです．

```cpp
#pragma omp parallel for
for(int i=0; i<N; i++){
    y[i] = 0;
    for(int j = row_ptr[i]; j < row_ptr[i+1]; j++){
        y[i] += val[j] * x[col_index[j]];
    }
}
```

### CRS形式のスレッド並列の問題点1 (スレッドごとの計算量の違い)
密行列と疎行列には明確に違う点があります，それは行ごとにいくつの要素が含まれているのかわからない点です．
ワーストケースとして，ある行だけ非零要素がN個あり，他の行は1つしかないような行列が考えられます(`「`←こういう埋まり方)．

このような行列への簡単な対処として，[OpenMPのスケジューリング方式](https://www.isus.jp/products/c-compilers/openmp-loop-scheduling/)を変更することが考えられます．
何も付けないと`static`になるのですが，とりあえず`guided`にしておくのが経験上おすすめです[参考](https://hishinuma-t.dev/papers/wo_review/hpcs2014/)．

ですが，これでも `「`みたいな形の埋まり方だと不均一がどうしても出ます．
そのような場合には，別の格納形式を検討する必要があるでしょう．それについては，またいずれ書きます．


## CRS形式とSIMD (未執筆)
未執筆です．
[SIMD演算を用いた高精度疎行列計算ソフトウェアの高速化](https://hishinuma-t.dev/papers/dr_thesis/) という素晴らしい論文があるという噂なので参考にしてください

以下，書きかけ：

行ごとにSIMDをかけていくことになります．
SIMDが分かっている人向けに，心の目で見ないと動かないAVXに似たようなSIMD width=4のときのコードを書いておきます．`reg`と名前にあるのがSIMDレジスタ型です:

```cpp
#pragma omp parallel for
for(int i=0; i<N; i++){
    y_reg = set_zero_pd(); // {0,0,0,0};
    for(int j = row_ptr[i]; j < row_ptr[i+1]; j+=4){
        A_reg = load_pd(val[j])
        x_reg = set_pd( 
            x[col_index[j+0]]
            x[col_index[j+1]]
            x[col_index[j+2]]
            x[col_index[j+3]]
            );

        y_reg = A_reg * x_reg
    }

    //行の要素数が4じゃないときのための端数処理がここに入る

    y[i] = y_reg[0] + y_reg[1] + y_reg[2] + y_reg[3]
}
```

吐き気がしてきましたね！Enjoy!

## 出典
CRSをrefしたいときはこれを使うとよい：

```
Richard Barrett, Michael W. Berry, Tony F. Chan, James Demmel, June Donato, Jack Dongarra, Victor Eijkhout, Roldan Pozo, Charles Romine and Henk A. van der Vorst, Templates for the Solution of Linear Systems: Building Blocks for Iterative Methods, SIAM: Philadelphia, PA, 1993.
```
