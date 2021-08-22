---
title: "Intel oneAPIのUbuntuへのインストールとToolkitのサイズでかすぎ問題"
type: "tech"
topics: ["cpp"]
published: true
---

# はじめに
これは [Intel oneAPIのIntelコンパイラやDPC++についてちょっと調べた](https://zenn.dev/hishinuma_t/articles/intel-oneapi_dpc) の番外編です．

この記事では2つの話を取り扱います．
1. Ubuntu 20.04.2 LTSへのoneAPIの導入 (CentOSへの導入については [別の記事](https://qiita.com/k_nitadori/items/84c2e7c6c1825c092cc9)があります)
2. 落としてくるとサイズが20GB以上あって辛すぎるから選んで入れようねという啓蒙活動

でかすぎ問題については，コンテナにして配布することを考えると，Cコンパイラが欲しいだけなのにコンテナが20GB超になるのはちょっと許せないです．
まとめて落ちてくるツールキットだと無駄なのが落ちてきまくるので，自分で選んで入れようねって話です．
なにが必要なのかは人によると思いますが，この記事では一例としてC/C++ユーザが使うことを考えてどうやってインストールしたら良いのかについて書きます．

ただaptで一個ずつ入れるだけの話だろ？と思うかもしれませんが，色々パッケージを見ていたら旧有償iccとは若干違いそうなところも見つけたので，色々書いてる．．．つもりです．

# Ubuntu 20.04.2 LTSへのoneAPIの導入

## リポジトリ登録
まぁどっかに書いてあるかもしれないけど，一応まとめておきます．

環境は `ubuntu:latest` コンテナを引っ張ってきてます．`20.04.2` です．

これは大した話ではなくて，ちゃんと[本家のマニュアル][1]があって，yumやらaptで入ることになってます．
本家のoneAPIコンテナはUbuntu18.04までしかありませんが，[システム要件のページ][2]を見ると，20.04でも入るということになっています．

[1]:https://software.intel.com/content/www/us/en/develop/documentation/installation-guide-for-intel-oneapi-toolkits-linux/top/installation/install-using-package-managers.html
[2]:https://software.intel.com/content/www/us/en/develop/documentation/installation-guide-for-intel-oneapi-toolkits-linux/top/installation/install-using-package-managers/apt.html#apt

ってことで雑なスクリプトを作りました．
私はコンテナで何も入ってないのでkeyの追加のためにwgetとgnupgを入れてますが，既にインストール済なら消すなりしてください

```sh
# use wget to fetch the Intel repository public key
apt update -y; apt install -y wget gnupg
cd /tmp
wget https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
# add to your apt sources keyring so that archives signed with this key will be trusted.
apt-key add GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
# remove the public key
rm GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
# Configure the APT client to use Intel’s repository:
echo "deb https://apt.repos.intel.com/oneapi all main" | tee /etc/apt/sources.list.d/oneAPI.list

apt update -y
```

これで `apt search` とかすれば見つかるはずです．

## インストール
iccとかvtuneとか，それぞれパッケージがあるけど，まとめたツールキットが配布されています．
公式でもこれをオススメしているようで，Qiitaとかの記事も探しましたが，皆これを入れているようです．

配布されているツールキットは以下の7つで，詳細は[公式のここ](https://software.intel.com/content/www/us/en/develop/tools/oneapi/all-toolkits.html#hpc-kit)に書いてあります．

- Intel® oneAPI Base Toolkit ( `apt install intel-basekit` )
- Intel® oneAPI HPC Toolkit ( `apt install intel-hpckit` )
- Intel® oneAPI AI Analytics Toolkit ( `apt install intel-aikit` )
- Intel® oneAPI IoT Toolkit ( `apt install intel-iotkit` )
- Intel® oneAPI Rendering Toolkit ( `apt install intel-renderkit` )
- Intel® oneAPI Distribution of OpenVINO Toolkit (私の環境では `apt search` で出てこず)
- Intel® System Bring-up Toolkit (私の環境では `apt search` で出てこず)

と思ったら， `apt search` に載っていない8つめが出てきました．[個別のページ](https://software.intel.com/content/www/us/en/develop/tools/oneapi/dl-framework-developer-toolkit.html#gs.8zbc3a)はあって，DeepLearning Frameworkだそうです．

- Intel® oneAPI DL Framework Developer Toolkit ( `apt install intel-dlfdkit` )

公式を見て好きなのを入れてください． 

＼＼＼ 解散 ／／／

と，思うじゃないですか．

# oneAPIさんのToolkitサイズでかすぎ問題
落とそうとしたときのメッセージを見てください．こんなのコンテナで配ってられません．

- basekit

```
1 upgraded, 81 newly installed, 0 to remove and 7 not upgraded.
Need to get 3292 MB of archives.
After this operation, 15.9 GB of additional disk space will be used.
```

- hpckit

```
1 upgraded, 100 newly installed, 0 to remove and 7 not upgraded.
Need to get 3578 MB of archives.
After this operation, 17.8 GB of additional disk space will be used.
```

実際にhpckitを入れるとディレクトリのサイズが22GBあって，ハゲ散らかります

```
root@610f1b12e48f:/# du -sh opt/intel/
22G     opt/intel/
```

そもそもDockerのコンテナもデカいのです．
```
intel/oneapi-hpckit  latest  68ffdbe86df4   7 weeks ago    22.7GB
```

参考までに，hpckitを入れようとすると出てくるパッケージリストは以下です．
`python` とかいう文字が見えます．iccが欲しいだけなのに余計なものを詰め込みやがってこのry．

```
 cmake cmake-data intel-basekit intel-basekit-getting-started intel-hpckit-getting-started intel-oneapi-advisor
  intel-oneapi-ccl-2021.3.0 intel-oneapi-ccl-devel intel-oneapi-ccl-devel-2021.3.0 intel-oneapi-clck
  intel-oneapi-clck-2021.3.0 intel-oneapi-common-licensing intel-oneapi-common-licensing-2021.3.0
  intel-oneapi-common-vars intel-oneapi-compiler-cpp-eclipse-cfg intel-oneapi-compiler-dpcpp-cpp
  intel-oneapi-compiler-dpcpp-cpp-2021.3.0 intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic
  intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic-2021.3.0
  intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic-common-2021.3.0
  intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic-runtime-2021.3.0
  intel-oneapi-compiler-dpcpp-cpp-classic-fortran-shared-runtime-2021.3.0
  intel-oneapi-compiler-dpcpp-cpp-common-2021.3.0 intel-oneapi-compiler-dpcpp-cpp-runtime-2021.3.0
  intel-oneapi-compiler-dpcpp-eclipse-cfg intel-oneapi-compiler-fortran intel-oneapi-compiler-fortran-2021.3.0
  intel-oneapi-compiler-fortran-common-2021.3.0 intel-oneapi-compiler-fortran-runtime-2021.3.0
  intel-oneapi-compiler-shared-2021.3.0 intel-oneapi-compiler-shared-common-2021.3.0
  intel-oneapi-compiler-shared-common-runtime-2021.3.0 intel-oneapi-compiler-shared-runtime-2021.3.0
  intel-oneapi-condaindex intel-oneapi-dal-2021.3.0 intel-oneapi-dal-common-2021.3.0
  intel-oneapi-dal-common-devel-2021.3.0 intel-oneapi-dal-daal4py-2021.3.0 intel-oneapi-dal-devel
  intel-oneapi-dal-devel-2021.3.0 intel-oneapi-dal-scikit-learn-intelex-2021.3.0 intel-oneapi-dev-utilities
  intel-oneapi-dev-utilities-2021.3.0 intel-oneapi-dev-utilities-eclipse-cfg intel-oneapi-dnnl
  intel-oneapi-dnnl-devel intel-oneapi-dpcpp-cpp-2021.3.0 intel-oneapi-dpcpp-ct intel-oneapi-dpcpp-ct-2021.3.0
  intel-oneapi-dpcpp-ct-eclipse-cfg intel-oneapi-dpcpp-debugger intel-oneapi-dpcpp-debugger-10.1.2
  intel-oneapi-dpcpp-debugger-eclipse-cfg intel-oneapi-inspector intel-oneapi-ipp-2021.3.0
  intel-oneapi-ipp-common-2021.3.0 intel-oneapi-ipp-common-devel-2021.3.0 intel-oneapi-ipp-devel
  intel-oneapi-ipp-devel-2021.3.0 intel-oneapi-ippcp-2021.3.0 intel-oneapi-ippcp-common-2021.3.0
  intel-oneapi-ippcp-common-devel-2021.3.0 intel-oneapi-ippcp-devel intel-oneapi-ippcp-devel-2021.3.0
  intel-oneapi-itac intel-oneapi-itac-2021.3.0 intel-oneapi-libdpstd-devel intel-oneapi-libdpstd-devel-2021.4.0
  intel-oneapi-mkl-2021.3.0 intel-oneapi-mkl-common-2021.3.0 intel-oneapi-mkl-common-devel-2021.3.0                                                    
  intel-oneapi-mkl-devel intel-oneapi-mkl-devel-2021.3.0 intel-oneapi-mpi-2021.3.0 intel-oneapi-mpi-2021.3.1                                           
  intel-oneapi-mpi-devel intel-oneapi-mpi-devel-2021.3.0 intel-oneapi-mpi-devel-2021.3.1                                                               
  intel-oneapi-onevpl-2021.4.0 intel-oneapi-onevpl-devel intel-oneapi-onevpl-devel-2021.4.0                                                            
  intel-oneapi-openmp-2021.3.0 intel-oneapi-openmp-common-2021.3.0 intel-oneapi-python intel-oneapi-tbb-2021.3.0                                       
  intel-oneapi-tbb-common-2021.3.0 intel-oneapi-tbb-common-devel-2021.3.0 intel-oneapi-tbb-devel                                                       
  intel-oneapi-tbb-devel-2021.3.0 intel-oneapi-vtune libarchive13 libdrm-common libdrm2 libjsoncpp1 libpciaccess0                                      
  librhash0 libssl-dev libssl1.1 libuv1 libxfixes3
```

## hpckitのなにが無駄なんや？

duしてみましょう．

```
4.7G    intelpython
3.7G    mkl
2.7G    compiler
2.6G    dal
1.6G    ipp
1.5G    vtune
1.4G    mpi

955M    advisor
800M    conda_channel
507M    dnnl
472M    itac
308M    inspector
238M    debugger
137M    clck
73M     ccl
63M     ippcp
63M     vpl
61M     dpcpp-ct
19M     dev-utilities
11M     tbb
2.2M    dpl

152K    licensing
28K     setvars.sh
28K     etc
16K     sys_check.sh
12K     modulefiles-setup.sh
8.0K    readme-get-started-linux-base-kit.html
8.0K    readme-get-started-linux-hpc-kit.html
4.0K    common.sh
4.0K    support.txt
```

intelpythonでかすぎ．．

この時点で，いらないディレクトリを手動で消すって思うかもしれませんが，困ったことに微妙にそれぞれ依存があって，適当に消すと全員死んだりします．

# oneAPIの個別のパッケージを見る

ということで，hpckitに頼らずにいるやつだけ入れていきます．C/C++コンパイラと，プロファイラと，MKLだけでいいでしょう．
なお，ハードウェアによって色々バイナリを切り替えてるっぽいので，ここに書いたサイズなどはあくまで参考値です．

ということで個別に入れるにはどうしたら良いのかを見ていきます．

## MKL入れる
OneAPIのMKLを入れるとcompilerの一部が降ってきます．

```
apt install intel-oneapi-mkl
```

```
The following NEW packages will be installed:
  intel-oneapi-common-licensing-2021.3.0 intel-oneapi-common-vars intel-oneapi-compiler-dpcpp-cpp-runtime-2021.3.0 intel-oneapi-compiler-shared-common-runtime-2021.3.0 intel-oneapi-compiler-shared-runtime-2021.3.0
  intel-oneapi-condaindex intel-oneapi-mkl intel-oneapi-mkl-2021.3.0 intel-oneapi-mkl-common-2021.3.0 intel-oneapi-openmp-2021.3.0 intel-oneapi-openmp-common-2021.3.0 intel-oneapi-tbb-2021.3.0
  intel-oneapi-tbb-common-2021.3.0
```

OpenMPやTBBに依存しているようです．DPC++のランタイムまで降ってくるのは何なんでしょうね？
これはもしかすると，これまでのMKLとは違うのかもしれません．

この時点で `opt/intel/oneapi/compiler/` が生まれますが，700MB程度の `bin/` のないライブラリだけの状態です．

## Cコンパイラ

### iccだけ入れたい (結論：不可能)

hpckitのパッケージ一覧を眺めると，iccっぽいのはこれですが，DPC++とくっついているようです．

```
  intel-oneapi-compiler-dpcpp-cpp-2021.3.0 
  intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic
  intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic-2021.3.0
  intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic-common-2021.3.0
  intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic-runtime-2021.3.0
```

とりあえずこれを入れようとすると: 

```
  apt install -y intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic
```

dpcppっぽいのがワンサカついてきます．うーん．．．

```
The following NEW packages will be installed:
  intel-oneapi-compiler-cpp-eclipse-cfg intel-oneapi-compiler-dpcpp-cpp-2021.3.0 intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic-2021.3.0
  intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic-common-2021.3.0 intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic-runtime-2021.3.0 intel-oneapi-compiler-dpcpp-cpp-classic-fortran-shared-runtime-2021.3.0
  intel-oneapi-compiler-dpcpp-cpp-common-2021.3.0 intel-oneapi-compiler-dpcpp-eclipse-cfg intel-oneapi-compiler-shared-2021.3.0 intel-oneapi-compiler-shared-common-2021.3.0 intel-oneapi-dev-utilities-2021.3.0
  intel-oneapi-dev-utilities-eclipse-cfg intel-oneapi-dpcpp-cpp-2021.3.0 intel-oneapi-dpcpp-debugger-10.1.2 intel-oneapi-dpcpp-debugger-eclipse-cfg intel-oneapi-libdpstd-devel-2021.4.0
  intel-oneapi-tbb-common-devel-2021.3.0 intel-oneapi-tbb-devel-2021.3.0
```

[前回の記事](https://zenn.dev/hishinuma_t/articles/intel-oneapi_dpc) のとおり，
この無償iccはOpenMPなどの一部のライブラリがDPC++のベースであるintel/LLVMに含まれるものを使っているので，
DPC++なしに入れることはできないようです．
つまりよくわかんないLLVMは必ず入ります．iccだけを入れることは不可能のようです．

### DPC++だけを入れる

iccだけを入れる方法はよくわかんないですが，DPC++を入れてみます．

```
apt install intel-oneapi-compiler-dpcpp-cpp
```

これはできます．classicは入りません．

```
The following NEW packages will be installed:
  intel-oneapi-compiler-cpp-eclipse-cfg intel-oneapi-compiler-dpcpp-cpp intel-oneapi-compiler-dpcpp-cpp-2021.3.0 intel-oneapi-compiler-dpcpp-cpp-common-2021.3.0 intel-oneapi-compiler-dpcpp-eclipse-cfg
  intel-oneapi-compiler-shared-2021.3.0 intel-oneapi-compiler-shared-common-2021.3.0 intel-oneapi-dev-utilities-2021.3.0 intel-oneapi-dev-utilities-eclipse-cfg intel-oneapi-dpcpp-cpp-2021.3.0
  intel-oneapi-dpcpp-debugger-10.1.2 intel-oneapi-dpcpp-debugger-eclipse-cfg intel-oneapi-libdpstd-devel-2021.4.0 intel-oneapi-tbb-common-devel-2021.3.0 intel-oneapi-tbb-devel-2021.3.0
```

パッケージの違いがほとんどありません．DPC++ってなんなんでしょうね．．．
僕にはiccにSYCLとOpenCLをかぶせた糖衣構文的なものにしか見えないんですけど，最適化のパスとかが違うんでしょうか．さっぱりわかりませんね．．

## プロファイラを入れる
ここでコンテナを一度作り直して，クリーンな状態にします．

vtuneだけあると便利なので，プロファイラだけを入れられるのかを見ていきたいと思います．

インストールは簡単で

```
apt install intel-oneapi-vtune
```

ほとんどvtuneだけ落としてこれます．これだけなら1.5GBで，他のものは落ちてきません．

```
The following NEW packages will be installed:
  intel-oneapi-common-licensing-2021.3.0 intel-oneapi-common-vars intel-oneapi-vtune
```

## [まとめ] ここまでを全部入れると
C/C++構成でよく使いそうなicc / dpcpp / mkl / vtuneを入れてみました．

```
apt install intel-oneapi-vtune intel-oneapi-compiler-dpcpp-cpp-and-cpp-classic intel-oneapi-mkl
```

duの結果は以下のとおりです．

```
2.5G    compiler
1.9G    mkl
1.5G    vtune

238M    debugger
119M    conda_channel
11M     tbb

268K    r000ps-bad
152K    licensing
140K    r001hs-bad
28K     setvars.sh
20K     etc
19M     dev-utilities
16K     sys_check.sh
12K     modulefiles-setup.sh
4.0K    support.txt
4.0K    common.sh
```

合計で6.2GBです．当初が22GBだったことを考えれば，使うやつだけ選べば随分減らせると思います．
追加でFortranコンパイラとかAnalyzerとか入れても10GBくらいじゃないでしょうか．

これらのことから，パッケージマネージャで入るようになったからってコンテナとかに愚直にhpckitを入れるとひどい目に合うので，
ちゃんと選んで入れるとかなり減らせるということが分かりました．

よかったよかった
