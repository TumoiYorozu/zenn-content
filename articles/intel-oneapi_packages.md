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

[1]:https://zenn.dev/hishinuma_t/articles/intel-oneapi_dpc
[2]:https://zenn.dev/hishinuma_t/articles/intel-oneapi_install
[3]:https://zenn.dev/hishinuma_t/articles/intel-oneapi_clang

私の認識に不足があり，上記の記事に対し以下の情報をいただきました．

https://twitter.com/subarutaro/status/1429836397754470408?s=20

私の記事ではicc/icpcに加えてLLVMベースのDPC++が増え，合計3つのコンパイラがあるという記述でしたが，DPC++はicx/icpxの拡張であり，実際にLLVMベースのコンパイラはicx/icpxというものでした．

整理しますと，
- 従来のインテルコンパイラ(classicと呼ばれる)であるiccとicpc
- LLVMベースの新しいインテルコンパイラであるicxとicpx
- icxとicpxにOpenCLやSYCLを加えたDPC++(コマンドとしてはdpcpp)

で，C/C++コンパイラが合計5種．

Fortranコンパイラとしてifortに加えて新しくifxが追加され，合計2種．

つまりoneAPIには，C/C++とFortranコンパイラをあわせて7つのコンパイラが用意されています．

先日の記事に書かせていただいた，DPC++がLLVMベースあることや，実験の結果に誤りはありませんが，情報が不足していた点，謹んでお詫びします．

本記事では，icx/icpxに加え，Fortranコンパイラも含めたoneAPIに含まれるC/C++/Fortranコンパイラについて整理します．
