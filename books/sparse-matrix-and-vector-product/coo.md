---
title: 疎行列の格納形式：COO
---
# 概要
COO形式 (Coordinate形式，座標形式)は疎行列の格納形式の中で最も理解しやすく，簡単で，人間に優しいフォーマットです．
つまり計算機に厳しいフォーマットです．

計算機に厳しい理由は2つで，(1) 次章で述べるCRS形式と比べてデータ量が多いこと，(2) COO形式で格納された疎行列に対して疎行列ベクトル積を行う場合，スレッド並列化が困難なことです．

繰り返しますが，**絶対に並列化が肝となるGPUなどでCOO形式を採用してはいけません**

[Scipy SparseのCOOのドキュメント ](https://docs.scipy.org/doc/scipy/reference/generated/scipy.sparse.coo_matrix.html#scipy.sparse.coo_matrix)

## hoge

![coo.png](https://raw.githubusercontent.com/t-hishinuma/zenn-content/main/books/sparse-matrix-and-vector-product/COO.png)
