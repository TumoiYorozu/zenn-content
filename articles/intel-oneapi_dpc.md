---
title: "Intel oneAPIのIntelコンパイラやDPC++についてちょっと調べた"
type: "tech"
topics: ["C/C++"]
published: false
---

# はじめに
もう1年くらい前からだが，oneAPIの登場に伴ってiccが無償化したという話があって，リポジトリをちょこっと追加するだけで簡単にIntel CompilerやIntel Vtuneなどが入るようになった [参考記事](https://qiita.com/k_nitadori/items/84c2e7c6c1825c092cc9)
Dockerでも使えるようになっており，[oneapi-hpckit](https://hub.docker.com/r/intel/oneapi-hpckit)を引っ張ってくればすぐに利用できる．簡単だね．

一方で，oneAPIには謎の [DPC++/C++コンパイラ](https://www.xlsoft.com/jp/products/intel/compilers/dpc/index.html) というのが入っている．
[公式](https://newsroom.intel.com/news/intels-one-api-project-delivers-unified-programming-model-across-diverse-architectures/#gs.9fgeo2)を見ると，Data Parallel C++という新しいプログラミング言語だとか書いてあって，それのコンパイラのようだ．

``
One API contains a new direct programming language, Data Parallel C++ (DPC++), an open, cross-industry alternative to single architecture proprietary languages. DPC++ delivers parallel programming productivity and performance using a programming model familiar to developers. DPC++ is based on C++, incorporates SYCL* from The Khronos Group and includes language extensions developed in an open community process.
```

これはLLVMベースのコンパイラとのことで，今までのIntel compilerとは完全に違いそうだ．
[DPC++はどうやらgithubで開発されてるっぽい？](https://github.com/intel/llvm)が，全てオープンなのかは知らない．

とはいえ中を読んでみると，ライブラリの拡張やOpenCL/SYCLが標準搭載されているだけのC++コンパイラに見える．iccにもOpenCLは付いてきたはずで，DPC++コンパイラとiccは何が違うのか？

そもそもiccはintel様の作った `/opt/intel/lib` みたいな所のライブラリを一部使ってくれてたはずで，oneAPIではどうなっているのか？

この記事では，とりあえず `-v` したり，`ldd`したり，ディレクトリを見たりして，表面的な違いをざっくり調べたので，記録を簡単にまとめる．
自動ベクトル化とか最適化周りについては，評価が難しいのでまた次回．

# 比較対象

- 過去の有償の製品版 Intel Compiler 
    - compilers_and_libraries_2020.4.304
    - 某スパコンに入っていたものなのでこれ以上は知らない
    - `icc -v`の結果: `icc バージョン 19.1.3.304 (gcc バージョン 4.8.5 互換)`

- oneAPIに含まれるIntel Compiler
    - intel/oneapi-hpckit:devel-ubuntu18.04コンテナに含まれるものを利用
    - `icc -v`の結果: icc version 2021.3.0 (gcc version 7.5.0 compatibility)
    - `which icc`の結果: `/opt/intel/oneapi/compiler/2021.3.0/linux/bin/intel64/icc`

- oneAPIに含まれるDPC++ 
    - intel/oneapi-hpckit:devel-ubuntu18.04コンテナに含まれるものを利用
    - `dpcpp -v`の結果は長かったので↓に
    - `which dpcpp`の結果: `/opt/intel/oneapi/compiler/2021.3.0/linux/bin/dpcpp`


`dpcpp -v` の結果
```
root@bf0951e94aef:/# dpcpp -v
Intel(R) oneAPI DPC++/C++ Compiler 2021.3.0 (2021.3.0.20210619)
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /opt/intel/oneapi/compiler/2021.3.0/linux/bin
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7.5.0
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/8
Selected GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7.5.0
Candidate multilib: .;@m64
Selected multilib: .;@m64
```

(参考までに) clang -vの結果
```
root@bf0951e94aef:/# clang -v
Intel(R) oneAPI DPC++/C++ Compiler 2021.3.0 (2021.3.0.20210619)
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /opt/intel/oneapi/compiler/2021.3.0/linux/bin
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7.5.0
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/8
Selected GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7.5.0
Candidate multilib: .;@m64
Selected multilib: .;@m64
```

clang乗っ取られとるやんけｗｗｗｗ

## わかったこと
- DPC++は[intel/LLVM](https://github.com/intel/llvm)をベースにして作られたLLVMのフロントエンドなので，ベースのLLVMがLLVM


# リンクされるライブラリ等

