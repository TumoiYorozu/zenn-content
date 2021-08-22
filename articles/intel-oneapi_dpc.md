---
title: "Intel oneAPIのIntelコンパイラやDPC++についてちょっと調べた"
type: "tech"
topics: ["cpp"]
published: true
---

# はじめに
もう1年くらい前からですが，oneAPIの登場に伴ってiccが無償化したという話があって，リポジトリをちょこっと追加するだけで簡単にIntel CompilerやIntel Vtuneなどが入るようになった [(参考記事)](https://qiita.com/k_nitadori/items/84c2e7c6c1825c092cc9)
Dockerでも使えるようになっており，[oneapi-hpckit](https://hub.docker.com/r/intel/oneapi-hpckit)を引っ張ってくればすぐに利用できる．簡単だね．

一方で，oneAPIには謎の [DPC++/C++コンパイラ](https://www.xlsoft.com/jp/products/intel/compilers/dpc/index.html) というのが入っている．
[公式](https://newsroom.intel.com/news/intels-one-api-project-delivers-unified-programming-model-across-diverse-architectures/#gs.9fgeo2)を見ると，Data Parallel C++という新しいプログラミング言語だとか書いてあって，それのコンパイラのようだ．

```
One API contains a new direct programming language, Data Parallel C++ (DPC++), 
an open, cross-industry alternative to single architecture proprietary languages.
DPC++ delivers parallel programming productivity and performance using a programming model familiar to developers.
DPC++ is based on C++, incorporates SYCL* from The Khronos Group and includes language extensions developed in an open community process.
```

これはLLVMベースのコンパイラで，今までのIntel compilerとは完全に違うっぽい．
[DPC++はどうやらgithubで開発されてるっぽい？](https://github.com/intel/llvm)が，全てオープンなのかは知らない．

とはいえ中を読んでみると，ライブラリの拡張やOpenCL/SYCLが標準搭載されているだけのC++コンパイラに見える．iccにもOpenCLは付いてきたはずで，DPC++コンパイラとiccは何が違うのか？

そもそもiccはintel様の作った `/opt/intel/lib` みたいな所のライブラリを一部使ってくれてたはずで，oneAPIではどうなっているのか？

この記事では，とりあえず `-v` したり，`ldd`したり，ディレクトリを見たりして，表面的な違いをざっくり調べたので，記録を簡単にまとめる．
最適化周りについては，評価が難しいのでまた次回．

# 比較対象

書き終わってから気づいたけどdpcppはC++なんだからicpcと比べるべきだったね（まぁだいたい同じだけども)．
- 過去の有償の製品版 Intel Compiler 
    - compilers_and_libraries_2020.4.304
    - `which icc`の結果: `/apps/intel/2020.4/compilers_and_libraries_2020.4.304/linux/bin/intel64/icc`
    - `icc -v`の結果: `icc バージョン 19.1.3.304 (gcc バージョン 4.8.5 互換)`
    - 某スパコンに入っていたものなのでこれ以上は知らない

- oneAPIに含まれるIntel Compiler
    - intel/oneapi-hpckit:devel-ubuntu18.04コンテナに含まれるものを利用
    - `which icc`の結果: `/opt/intel/oneapi/compiler/2021.3.0/linux/bin/intel64/icc`
    - `icc -v`の結果: `icc version 2021.3.0 (gcc version 7.5.0 compatibility)`

- oneAPIに含まれるDPC++ 
    - intel/oneapi-hpckit:devel-ubuntu18.04コンテナに含まれるものを利用
    - `which dpcpp`の結果: `/opt/intel/oneapi/compiler/2021.3.0/linux/bin/dpcpp`
    - `dpcpp -v`の結果は長かったので↓に


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

あれ，clangがDPCと全く一緒なんだが？
なるほどLLVMに手を入れてintel/LLVMを作っているからフロントエンドのclangとDPC++は両方作れるわけで，このコンテナではバックエンドはintel/LLVMのclang, dpcppになっているのか．．．

そしてiccもDPC++も `/opt/intel/oneapi/compiler/` の下にあることもわかった．
もしかしてライブラリは共通か．．？

# リンクされるライブラリ等
適当なコードでOpenMPと数学ライブラリを使うようにして，`-O3` と `-fopenmp` と `-lm` をつけたものを `ldd` してみた．

## 有償icc (19.1.3.304)

```
	linux-vdso.so.1 =>  (0x00007fff48383000)
	libm.so.6 => /lib64/libm.so.6 (0x00007f979e59b000)
	libstdc++.so.6 => /lib64/libstdc++.so.6 (0x00007f979e293000)
	libiomp5.so => /apps/intel/2020.4/compilers_and_libraries_2020.4.304/linux/compiler/lib/intel64_lin/libiomp5.so (0x00007f979de8d000)
	libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007f979dc77000)
	libpthread.so.0 => /lib64/libpthread.so.0 (0x00007f979da5b000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f979d68e000)
	libdl.so.2 => /lib64/libdl.so.2 (0x00007f979d48a000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f979e89d000)
```

libiomp5.soが `/opt/intel` をちゃんと見てる．それ以外は `lib64`．

あれ，mathも汎用が使われるんだっけ？コンパイラオプションでintel mathに変わる？忘れた．．．

`-march=native` とか `Ofast` とか付けてみたけどlddの結果に変化はなかった．

## oneAPIに含まれる無償icc (2021.3.0)

```
	linux-vdso.so.1 (0x00007ffcf1bf1000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f0f22e1f000)
	libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f0f22a96000)
	libiomp5.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libiomp5.so (0x00007f0f22681000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f0f22462000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f0f22071000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f0f21e6d000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f0f231bd000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f0f21c55000)
	librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f0f21a4d000)
```

libiomp5.soが `/opt/intel/oneapi/` を見てる．それ以外は `/lib/x86_64_-linux-gnu/`．

OpenMPについてはintel OpenMPを使う模様．最適化の出来は不明だが，これは商用iccが完全に無償になったものと捉えて良さそう？

## DPC++
```
	linux-vdso.so.1 (0x00007ffd3af7c000)
	libimf.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libimf.so (0x00007fbd78dbb000)
	libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007fbd78a1d000)
	libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007fbd78694000)
	libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007fbd7847c000)
	libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fbd78278000)
	libiomp5.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libiomp5.so (0x00007fbd77e63000)
	libsycl.so.5 => /opt/intel/oneapi/compiler/2021.3.0/linux/lib/libsycl.so.5 (0x00007fbd77baa000)
	libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fbd7798b000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fbd7759a000)
	libintlc.so.5 => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libintlc.so.5 (0x00007fbd77322000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fbd79443000)
	librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007fbd7711a000)
	libOpenCL.so.1 => /opt/intel/oneapi/compiler/2021.3.0/linux/lib/libOpenCL.so.1 (0x00007fbd79652000)
	libsvml.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libsvml.so (0x00007fbd75615000)
	libirng.so => /opt/intel/oneapi/compiler/2021.3.0/linux/compiler/lib/intel64_lin/libirng.so (0x00007fbd752ab000)
```

libiomp5.soが `/opt/intel/oneapi/` を見てる．つまりこの辺はiccと共通らしい．
それ以外は `/lib/x86_64_-linux-gnu/`で，これもiccと同じ．

一方でSYCLとかOpenCLを頼んでもいないのに勝手にリンクするようになった．クロスコンパイルするときにすごく面倒臭そう．

あと`libimf.so`とか`libintlc.so.5`とかがリンクされるようになった．

`nm`でみてみると，`libimf.so`は数学関数，`libintlc.so`は`intel_memcpy`みたいなシンボルが見えたので，icc専用の高速な実装だろうか？
iccだとリンクされなかったんだが，リンク条件が変わったのか？

なんか邪魔くさいのがいっぱい付いてきたけど，intel専用のライブラリをうまく使ってくれているんだとしたら，こっちのほうが速くなりそうな感じはある．
あとはバックエンドがintel独自からLLVM拡張になったことでどうなるか？

ただ，従来のiccを使うか，dpcppを使うかについては，性能以前にSYCLを外せるかどうか，クロスコンパイルできるかどうかなどを調べてからのほうが良さそうだ．

# OpenMP ライブラリ周り

コンパイラ周辺にあるライブラリにずいぶん違いがあったので，書いておく．

## 有償icc
```
cilk_db.so         libchkpwrap_w.a  libifcoremt.a      libintlc.so       libirc.a        libpdbx.so
crt                libcilkrts.so    libifcoremt.so     libintlc.so.5     libirc.so       libpdbx.so.5
for_main.o         libcilkrts.so.5  libifcoremt.so.5   libiomp5.a        libirc_s.a      libpdbxinst.a
init.o             libdecimal.a     libifcoremt_pic.a  libiomp5.dbg      libirng.a       libqkmalloc.so
libbfp754.a        libicaf.so       libifport.a        libiomp5.so       libirng.so      libsvml.a
libchkp.so         libifcore.a      libifport.so       libiomp5_db.so    libistrconv.a   libsvml.so
libchkpwrap.a      libifcore.so     libifport.so.5     libiompstubs5.a   libistrconv.so  locale
libchkpwrap_h.a    libifcore.so.5   libimf.a           libiompstubs5.so  libmatmul.a     
libchkpwrap_h_w.a  libifcore_pic.a  libimf.so          libipgo.a         libpdbx.a       
```

ここに `libintlc.so` あるのに，さっきなんでリンクされなかったんだ？
やっぱりオプションの条件が違うのかな．．．

## OneAPI
```
clang                             libomp-fallback-complex.o       libsycl-fallback-cassert.spv
clbltfnshared.rtl                 libomp-fallback-complex.spv     libsycl-fallback-cmath-fp64.o
emu                               libomp-glibc.o                  libsycl-fallback-cmath-fp64.spv
icx-lto.so                        libomp-itt-compiler-wrappers.o  libsycl-fallback-cmath.o
libOpenCL.so                      libomp-itt-stubs.o              libsycl-fallback-cmath.spv
libOpenCL.so.1                    libomp-itt-user-wrappers.o      libsycl-fallback-complex-fp64.o
libOpenCL.so.1.2                  libomp-spirvdevicertl.o         libsycl-fallback-complex-fp64.spv
libfortran-target.o               libomptarget-opencl.bc          libsycl-fallback-complex.o
libomp-cmath-fp64.o               libomptarget.rtl.level0.so      libsycl-fallback-complex.spv
libomp-cmath.o                    libomptarget.rtl.opencl.so      libsycl-itt-compiler-wrappers.o
libomp-complex-fp64.o             libomptarget.rtl.x86_64.so      libsycl-itt-stubs.o
libomp-complex.o                  libomptarget.so                 libsycl-itt-user-wrappers.o
libomp-fallback-cassert.o         libpi_level_zero.so             libsycl.so
libomp-fallback-cassert.spv       libpi_opencl.so                 libsycl.so.5
libomp-fallback-cmath-fp64.o      libsycl-cmath-fp64.o            libsycl.so.5.2.0
libomp-fallback-cmath-fp64.spv    libsycl-cmath.o                 libsycl.so.5.2.0-gdb.py
libomp-fallback-cmath.o           libsycl-complex-fp64.o          oclfpga
libomp-fallback-cmath.spv         libsycl-complex.o               sycl.conf
libomp-fallback-complex-fp64.o    libsycl-crt.o                   x64
libomp-fallback-complex-fp64.spv  libsycl-fallback-cassert.o
```

libompが完全にLLVMになって，たくさん出てくるようになった．あとはsycl系か．
しかし，現段階では少なくとも `libomptarget-nvptx` はないので，NVIDIA GPU用のバイナリは出さないだろう(それはそうだと思ってたが)

CUDAのある環境にoneAPIを入れようとすると見つけてくるのかもしれない？
NVIDIAコンテナにoneAPIリポジトリを追加してやれば作ってくれる？
LLVMからビルドし直さないと駄目？

# 今後の課題
どうもライブラリはiccとDPC++で共通っぽいが，オプションとか色々違っててよくわからない．

`libintel` とか勝手にリンクされたけど，intelじゃなくてAMDのCPUに入れようとしたりするとどうなるのかについて調べたいところ．

最適化のパスは違うんだろうと思うので，性能についても検証したい．

`/ope/oneapi` には，以下のようにvtuneとかmpiもあるので，この辺もちゃんと見ておきたい．

```
root@bf0951e94aef:/opt/intel/oneapi# ls
advisor    conda_channel  dpl          ippcp                 mpi                                     sys_check.sh
ccl        dal            etc          itac                  readme-get-started-linux-base-kit.html  tbb
clck       debugger       inspector    licensing             readme-get-started-linux-hpc-kit.html   vpl
common.sh  dev-utilities  intelpython  mkl                   setvars.sh                              vtune
compiler   dnnl           ipp          modulefiles-setup.sh  support.txt
```

# 追記
intel/LLVMをバックエンドにしたclangがあるんだったらDPC++なんて使わなくていいじゃんって思ったんだ

```
root@c36a8f3d4268:/# clang++ -fopenmp a.cpp 
a.cpp:1:9: fatal error: 'omp.h' file not found
#include<omp.h>
        ^~~~~~~
1 error generated.
```

( ﾟдﾟ)ハァ？
