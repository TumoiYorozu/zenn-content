---
title: "Intel oneAPIに含まれるC/C++/Fortranコンパイラについて"
type: "tech"
topics: ["cpp"]
published: true
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

私の記事ではicc/icpcに加えてLLVMベースのDPC++が増え，合計3つのコンパイラがあるという記述でしたが，実際にIntelコンパイラをLLVMベースとして作り直したものの名前はicx/icpxというものでした．

DPC++はicx/icpxの拡張であり，SYCLなどを有効にしたものになります．

つまり，先日の記事に書かせていただいた，DPC++がLLVMベースあることや，実験の結果に誤りはありませんが，情報が不足して居たということになります．この点，謹んでお詫びします．

調べ直してきまして，ざっくり整理しますと，
- 従来のインテルコンパイラ(Intel Compiler classic)であるiccとicpc
- LLVMベースの新しいインテルコンパイラであるicxとicpx
- icxとicpxにOpenCLやSYCLを加えたDPC++(コマンドとしてはdpcpp)
- [Intel/LLVM](https://github.com/intel/llvm) に含まれるClang

で，C/C++コンパイラが合計6種．

Fortranコンパイラとしてifortに加えて新しくifxが追加され，合計2種．

つまりoneAPIには，C/C++とFortranコンパイラをあわせて8つのコンパイラが用意されています．

本記事では，icx/icpxに加え，Fortranコンパイラも含めたoneAPIに含まれるC/C++/Fortranコンパイラについて整理します．

# Intelのコンパイラ開発ロードマップ
[こちらの動画](https://techdecoded.intel.io/essentials/introducing-the-next-gen-of-intel-parallel-studio-transitioning-to-the-latest-hpc-software-development-suite/#gs.9h029u)の24分頃に，Intelのコンパイラに関するロードマップが紹介されています．

![intel_compiler_roadmap.png](https://raw.githubusercontent.com/t-hishinuma/zenn-content/main/articles/img/intel_compiler_roadmap.png)

この画像から，以下のことが分かります．

- 従来のicc/icpcは2022年末，ifortは2023年末をもってレガシー扱いになる
- icx/icpx/dpcppは既にリリース済
- ifxは現在はベータ版で，2022年にリリースされる

それぞれがoneAPIの[oneapi-hpckit](https://hub.docker.com/r/intel/oneapi-hpckit)においてどういう状況下を確認していきます．

# icc/icpc/ifort
これは従来のIntelコンパイラということになっています．ただ，過去の記事でも少し触れたとおり，OpenMPなどのライブラリはicx/icpxと共通であり，完全な後継品なのかは不明です．

```sh
# icc -v
icc version 2021.3.0 (gcc version 7.5.0 compatibility)

# icpc -v
icpc version 2021.3.0 (gcc version 7.5.0 compatibility)

# ifort -v
ifort version 2021.3.0
```

前述の通り，icc/icpcは2022年末，ifortは2023年末をもってレガシー扱いになるようです．

## リンクされるライブラリ
適当なコードに， `-O3 -fopenmp -lm` をつけてコンパイルして `ldd` してみると， `libiomp5.so` がIntelのライブラリですね．

```
# ldd a.out
        linux-vdso.so.1 (0x00007ffd50b18000)
        libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f36e6836000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f36e6498000)
        libiomp5.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libiomp5.so (0x00007f36e6083000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f36e5e64000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f36e5a73000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f36e586f000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f36e6bbf000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f36e5657000)
        librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f36e544f000)
```

# icx/icpx
こちらが新しいLLVMベースのコンパイラです．
[Intel/LLVM](https://github.com/intel/llvm) をビルドすれば得られるらしいですが，自前ビルドはまだ試してないです．

```sh
# ls
a.cpp  a.out  a.sh
root@0216429c2b14:/work# icx -v
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

# icpx -v
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

## リンクされるライブラリ
適当なコードに， `-O3 -fopenmp -lm` をつけてコンパイルして `ldd` してみると， 
- `libiomp5.so` (OpenMP)
- `libimf.so` (数学関数)
- `libintlc.so.5` (メモリ操作などの最適化ライブラリ)

がリンクされます．iccよりもIntel特製をいっぱい引っ張ってきました．速いのかは試してないですが．．．

```
root@0216429c2b14:/work# ldd a.out
        linux-vdso.so.1 (0x00007ffef8462000)
        libimf.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libimf.so (0x00007f8c79d62000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f8c799c4000)
        libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f8c7963b000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f8c79423000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f8c7921f000)
        libiomp5.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libiomp5.so (0x00007f8c78e0a000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f8c78beb000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f8c787fa000)
        libintlc.so.5 => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libintlc.so.5 (0x00007f8c78582000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f8c7a3ea000)
        librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f8c7837a000)
```

# DPC++
これはSYCLやOpenCLに対応したことでGPUやFPGAに向けたコードを開発できる．．．ことになっているoneAPIのC++コンパイラです．

DPC++に対応するCやFortranのコンパイラはありません．

以下の `dpcpp -v` の結果を見る限り，icx/icpxと同じメッセージになっています．

```
# dpcpp -v
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

## リンクされるライブラリ
適当なコードに， `-O3 -fopenmp -lm` をつけてコンパイルして `ldd` してみると，icx/icpxのときのものに加えて，
- `libOpenCL.so.1` (OpenCL)
- `libsycl.so` (SYCL)
- `libsvml.so` (Short Vector Mathematical Library)
- `libirng.so` (乱数)

を勝手に引っ張ってきました．勝手な想像ですが，DPC++はicx/icpxの糖衣構文で，リンクオプション等を色々くっつけているだけのものではないでしょうか．

```sh
# ldd a.out
        linux-vdso.so.1 (0x00007ffcc8d79000)
        libimf.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libimf.so (0x00007f1773b33000)
        libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f1773795000)
        libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f177340c000)
        libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f17731f4000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f1772ff0000)
        libiomp5.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libiomp5.so (0x00007f1772bdb000)
        libsycl.so.5 => /opt/intel/oneapi/compiler/2021.3.0/linux/lib/libsycl.so.5 (0x00007f1772922000)
        libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f1772703000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f1772312000)
        libintlc.so.5 => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libintlc.so.5 (0x00007f177209a000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f17741bb000)
        librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f1771e92000)
        libOpenCL.so.1 => /opt/intel/oneapi/compiler/2021.3.0/linux/lib/libOpenCL.so.1 (0x00007f17743ca000)
        libsvml.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libsvml.so (0x00007f177038d000)
        libirng.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libirng.so (0x00007f1770023000)
```

# intel/LLVMベースのClang
これについてはよくわからないですが，[前回の記事][3] で結構書いたので省略します．

# ifx
これはまだBetaということらしいですが入っています．おかしいLLVM/flangなんてものは幻のはずだ．．．！

それではヘルプを見てみましょう．

```sh
# ifx -help




                         Intel(R) Fortran Compiler Help
                         ==============================


  Intel(R) Compiler includes compiler options that optimize for instruction
  sets that are available in both Intel(R) and non-Intel microprocessors, but
  may perform additional optimizations for Intel microprocessors than for
  non-Intel microprocessors. In addition, certain compiler options for
  Intel(R) Compiler are reserved for Intel microprocessors. For a detailed
  description of these compiler options, including the instructions they
  implicate, please refer to "Intel(R) Compiler User and Reference Guides >
  Compiler Options."

  usage: ifort -qnextgen [options] file1 [file2 ...]
```

いいですか？

```sh
  usage: ifort -qnextgen [options] file1 [file2 ...]
```

はい．

一応バイナリは別物らしいけど．．．なにが違うのかわからないですね．．

```
# diff ifx intel64/ifort
Binary files ifx and intel64/ifort differ
# du ifx
3184    ifx
# du intel64/ifort
4140    intel64/ifort
```
