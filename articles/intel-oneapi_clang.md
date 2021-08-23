---
title: "Intel oneAPIを入れるとClangは死ぬ"
type: "tech"
topics: ["cpp"]
published: true
---

# はじめに
**TL;DR. Intel oneAPIとClangは競合する可能性があります**

iccが無償になって，yumやaptで簡単に入るよって書いてる記事が増えてきました．
気軽な気持ちで入れられるような顔をしていますが，これは罠です． `setvars.sh` とかいうクソスクリプトがシステムの環境変数設定をぶち壊す可能性があります．

自分のマシンなら良いですが，何も考えずに共用のマシンにインストールするのは考え直してください．

この記事は以下2つの記事の続きです．何のためにインストール記事なんて書いたかって，実はこの記事をちゃんと書きたかったからです．
- [Intel oneAPIのIntelコンパイラやDPC++についてちょっと調べた][1]
- [Intel oneAPIのUbuntuへのインストールとToolkitのサイズでかすぎ問題][2]

[1]:https://zenn.dev/hishinuma_t/articles/intel-oneapi_dpc
[2]:https://zenn.dev/hishinuma_t/articles/intel-oneapi_install

## 忙しい人のためのまとめ
- oneAPIからDPC++というLLVMベースのC++コンパイラが入りました．
    - ベースのLLVMは通常のものではなく，[IntelによるLLVMのフォーク品](https://github.com/intel/llvm) (intel/LLVM)です
- これはiccやMKLを入れようとすると強制でくっついてきます ([前回の記事][2])
    - [余談] どうやらDPC++とiccのライブラリは一部共通のようで，従来までの有償iccと同じ系譜のものなのか，よく分かりません
- intel/LLVMをインストールさせられた結果，通常のLLVMと環境変数やライブラリがぶつかります
- 良く分からない人が気軽に共用サーバなどに入れるべきではないでしょう

まぁ，同じシステムに2つコンパイラを入れた時点でリンクが崩壊するのは当たり前なのですが，今までのiccでは起きなかったことなので，覚えておいたほうが良いでしょう．

と，いうことで，以下では環境変数を乗っ取られて頭がおかしくなっていくclangくんの様子を観察したいと思います．

# 普通に環境構築

ベース環境は `ubuntu:latest` コンテナを引っ張ってきてます．`20.04.2` です．

## clangのインストール

既にclangが入ってるシステムを想定し，まずclangをインストールしておきます．
私は諸事情によりソースからビルドしていますが，なんでも良いと思います．

```sh
# which clang
/usr/local/llvm-12.0.1/bin/clang

# clang -v
clang version 12.0.1
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /usr/local/llvm-12.0.1/bin
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7.5.0
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/8
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/9
Selected GCC installation: /usr/lib/gcc/x86_64-linux-gnu/9
Candidate multilib: .;@m64
Selected multilib: .;@m64
```

## oneAPIのインストール
[前回の記事][2]どおりにoneAPIを入れます．
パッケージを選んで入れたほうがいいですが，今回は面倒なのでhpckitを突っ込みます．

この時点でDPC++とiccが両方入ります．
DPC++は単独で入れられますが，iccは駄目です．必ずDPC++が必要です (これも[前回の記事][2]に書きました)．
そのほか，vtuneとかだけなら単独でもいけますが，MKLはTBBとDPC++の一部 (OpenMP)を引っ張ってきます．

インストールが終わったら， `setvars.sh` を使ってパスを通しましょう！

```sh
apt install -y intel-hpckit
. /opt/intel/oneapi/setvars.sh
```

- iccが入りました

```sh
# which icc
/opt/intel/oneapi/compiler/2021.3.0/linux/bin/intel64/icc

# icc -v
icc version 2021.3.0 (gcc version 9.3.0 compatibility)
```

- DPC++が入りました

```sh
# which dpcpp
/opt/intel/oneapi/compiler/2021.3.0/linux/bin/dpcpp

# dpcpp -v
Intel(R) oneAPI DPC++/C++ Compiler 2021.3.0 (2021.3.0.20210619)
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /opt/intel/oneapi/compiler/2021.3.0/linux/bin
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/9
Selected GCC installation: /usr/lib/gcc/x86_64-linux-gnu/9
Candidate multilib: .;@m64
Selected multilib: .;@m64
```

普通の記事ならここで
「インストールできました．いかがでしたか？」
と締めるところですが，ここからが本題です．

# おや．．．？clangの様子が．．．
おめでとう！clangはintel/LLVMのclangに進化した！

```sh
# which clang
/opt/intel/oneapi/compiler/2021.3.0/linux/bin/clang

# clang -v
Intel(R) oneAPI DPC++/C++ Compiler 2021.3.0 (2021.3.0.20210619)
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir: /opt/intel/oneapi/compiler/2021.3.0/linux/bin
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/7.5.0
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/8
Found candidate GCC installation: /usr/lib/gcc/x86_64-linux-gnu/9
Selected GCC installation: /usr/lib/gcc/x86_64-linux-gnu/9
Candidate multilib: .;@m64
Selected multilib: .;@m64
```

は？？？？？？

おわかりでしょうか？ `/opt/intel` にある謎のclangが見えています．つまりこれはintel/LLVMのclangです．
intel/LLVMのもつclangにパスが通され， `/usr/bin/clang` は外されました．

パスも全滅です．

## 動かしてみる

試しに，これで適当なOpenMPとか数学関数を使うコード用意して， `-O3 -fopenmp -lm` をつけてコンパイルして `ldd` してみます．

```sh
# ldd a.out 
        linux-vdso.so.1 (0x00007ffd3ad4c000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f1a2d747000)
        libiomp5.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libiomp5.so (0x00007f1a2d332000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f1a2d30f000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f1a2d11b000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f1a2d89f000)
        librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f1a2d110000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f1a2d0f5000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f1a2d0ef000)
```

なんとintel/LLVMさんの `libiomp5.so` を引っ張ってきました．ちょっと待って勘弁してください．
これはclangに対して期待される動作ではありません．．．

こんなもので 
「clangで性能評価したらiccと変わりませんでした☆いかがでしたか？」
とかいうブログを書かれたら柱に頭を打ち付けて死ぬしかありません．

## ええい．古い方のclangを直接指定だ

駄目です．oneapiの方を見ています．何なんでしょうかこれ？

```sh
# ldd a.out 
        linux-vdso.so.1 (0x00007ffee814a000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f17cb25b000)
        libiomp5.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libiomp5.so (0x00007f1a2d332000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f17cb0fa000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f17caf06000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f17cb3b3000)
        librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f17caefb000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f17caef5000)

```

どうやらパスが全部 `/opt/intel` に持っていかれているようです．

```sh
# echo $LIBRARY_PATH
/opt/intel/oneapi/vpl/2021.4.0/lib:/opt/intel/oneapi/tbb/2021.3.0/env/../lib/intel64/gcc4.8:/opt/intel/oneapi/mpi/2021.3.1//libfabric/lib:/opt/intel/oneapi/mpi/2021.3.1//lib/release:/opt/intel/oneapi/mpi/2021.3.1//lib:/opt/intel/oneapi/mkl/2021.3.0/lib/intel64:/opt/intel/oneapi/ippcp/2021.3.0/lib/intel64:/opt/intel/oneapi/ipp/2021.3.0/lib/intel64:/opt/intel/oneapi/dnnl/2021.3.0/cpu_dpcpp_gpu_dpcpp/lib:/opt/intel/oneapi/dal/2021.3.0/lib/intel64:/opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin:/opt/intel/oneapi/compiler/2021.3.0/linux/lib:/opt/intel/oneapi/clck/2021.3.0/lib/intel64:/opt/intel/oneapi/ccl/2021.3.0/lib/cpu_gpu_dpcpp

# echo $CPATH
/opt/intel/oneapi/vpl/2021.4.0/include:/opt/intel/oneapi/tbb/2021.3.0/env/../include:/opt/intel/oneapi/mpi/2021.3.1//include:/opt/intel/oneapi/mkl/2021.3.0/include:/opt/intel/oneapi/ippcp/2021.3.0/include:/opt/intel/oneapi/ipp/2021.3.0/include:/opt/intel/oneapi/dpl/2021.4.0/linux/include:/opt/intel/oneapi/dpcpp-ct/2021.3.0/include:/opt/intel/oneapi/dnnl/2021.3.0/cpu_dpcpp_gpu_dpcpp/lib:/opt/intel/oneapi/dev-utilities/2021.3.0/include:/opt/intel/oneapi/dal/2021.3.0/include:/opt/intel/oneapi/compiler/2021.3.0/linux/include:/opt/intel/oneapi/ccl/2021.3.0/include/cpu_gpu_dpcpp
```

これを書き換えたら上手く行くんでしょうけど，そうしたらoneAPIが死ぬ気がします．．．．
ちょっとやる気が起きません．

この辺の環境変数設定がどうなってるのかよくわからないし，入れ方を変えたり適切に設定したらなんか上手く行くのかもしれませんが，1つの環境に2つのLLVMとclangがインストールされてしまうため，なんだか良く分からない事故は常に起こりそうです．
自分のマシンなら良いとは思いますが，共有サーバに入れるのはやめたほうがいいかもしれません．

前述の通りDPC++を避けてインストールできるのはvtuneなどのプロファイラだけなので，なんだかなあという感じですね．

# [補足] ちなみに，後から入れたら？

`intel/oneapi-hpckit:latest` コンテナに，後からaptでclangを入れてみることにします．

ちなみに，コンテナの初期状態では `/opt/intel` のclangが見えてます．

```sh
which clang
/opt/intel/oneapi/compiler/2021.3.0/linux/bin/clang

# clang -v
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

こいつでさっきのコードをコンパイルして `ldd` すると，今度は `/opt/intel` と `/usr/bin/` のどちらのclangも `/usr/lib` のclangを見ます．

```sh
# which clang
/opt/intel/oneapi/compiler/2021.3.0/linux/bin/clang

# clang -O3 -fopenmp -lm a.cpp; ldd ./a.out
        linux-vdso.so.1 (0x00007ffe007e8000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f787b0d2000)
        libomp.so.5 => /usr/lib/x86_64-linux-gnu/libomp.so.5 (0x00007f787ae1d000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f787abfe000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f787a80d000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f787b470000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f787a609000)

# /usr/bin/clang -O3 -fopenmp -lm a.cpp; ldd ./a.out
        linux-vdso.so.1 (0x00007ffe007fe000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f1131a0d000)
        libomp.so.5 => /usr/lib/x86_64-linux-gnu/libomp.so.5 (0x00007f1131758000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f1131539000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f1131148000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f1131dab000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f1130f44000)
```

これはこれでclangはclangのを見てくれているので正しい気がしますが， `/opt/intel` のclangは intel/LLVMのライブラリを見てほしい気がするので，やっぱりどうしようもないなって感じです．
