---
title: "Intel oneAPIに含まれるC/C++/Fortranコンパイラについて"
type: "tech"
topics: ["cpp"]
published: false
---

# はじめに
この記事は以下3つの記事の続きです．
- [Intel oneAPIのIntelコンパイラやDPC++についてちょっと調べた][1]
- [Intel oneAPIのUbuntuへのインストールとToolkitのサイズでかすぎ問題][2]
- [Intel oneAPIを入れるとClangが死ぬかもしれない][3]

上記3記事ですが，私の認識に不足があり，以下の情報をいただきました．

https://twitter.com/subarutaro/status/1429849613909909504?s=20

私の記事ではicc/icpcとLLVMベースのDPC++しかないような記述でしたが，DPC++はicx/icpxの拡張であり，実際にLLVMベースのコンパイラはicx/icpxというものでした．
DPC++がLLVMベースあることや，実験の結果に誤りはありませんが，情報が不足していた点，謹んでお詫びします．

本記事では，icx/icpxに加え，Fortranコンパイラも含めたoneAPIに含まれるC/C++/Fortranコンパイラについて整理します．
