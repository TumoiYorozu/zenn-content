---
title: 疎行列の格納形式：COO
---
# 概要
COO形式 (Coordinate形式，座標形式)は疎行列の格納形式の中で最も理解しやすく，簡単で，人間に優しいフォーマットです．
つまり計算機に厳しいフォーマットです．

計算機に厳しい理由は2つで，(1) 次章で述べるCRS形式と比べてデータ量が多いこと，(2) COO形式で格納された疎行列に対して疎行列ベクトル積を行う場合，スレッド並列化が困難なことです．

そのため行列の作成や，Matrix Market Formatなどの形式でのファイルへの読み書きに使われることが多い．


[Scipy SparseのCOOのドキュメント ](https://docs.scipy.org/doc/scipy/reference/generated/scipy.sparse.coo_matrix.html#scipy.sparse.coo_matrix)

## COO formatの構成

以下にCOO形式のメモリレイアウトを示します．

![coo.png](https://raw.githubusercontent.com/t-hishinuma/zenn-content/main/books/sparse-matrix-and-vector-product/COO.png)

# 余談，それでもCOO形式を使いたい場合
COOの問題点は行単位で計算することができず，単純な実装では並列化時にスレッドセーフにならないことです．
それでもCOO形式を並列化したい場合は，あらかじめ行列を行を跨がないようにブロック行分割しておくことになります．

流体解析でよく使われる [OpenFOAM](https://www.openfoam.com/) は，格納形式にCOOを採用しています．OpenFOAMの並列化は主にプロセス並列で，行列を予めプロセスごとに分割して保持するためCOOでも並列化できています．

ただし，GPUなどの数千コアのプロセッサだと，行列を数千に分割し，数千プロセスを立ち上げねばなならず現実的ではありません．
やはり現代においては理由がない限りCOO形式は計算に用いるべきではないでしょう．
