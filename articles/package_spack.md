---
title: "パッケージ配布について考える2：spack"
type: "tech"
topics: ["Tech", "C++"]
published: true
---

# はじめに
パッケージマネージャspackについて調べる．

spack Link: https://github.com/spack/spack

前回の記事はこれ：https://zenn.dev/hishinuma_t/articles/package_mamba

## TL; DR
spackはパッケージマネージャです．
ローレンス・リバモア主体で作ってて，独自アーキ/コンパイラが入り交じるのスパコンの闇を解決したいらしい．
**なので基本的にソースコードを落としてきて全部ビルドする思想です**

spackを `git clone` して `bin/spack` を叩くだけで動きます
CUDAなんかは，"ビルド"と称してバイナリ落としてきて展開だけしてるみたい．

spackコマンドの中身はpythonスクリプトで，各ソフトの用意した `package.py` に従ってダウンロード＆ビルドします．
なのでパスとか通さない限り，`bin/spack` を移動すると 相対位置がずれて`package.py` が見つからないので死にます．

pythonコードが作り込まれてて，うまくmakeとかcmakeとかのオプションを配列にするだけでやってくれるっぽい．ぜったい嘘
色々やってみたけど，物によっては当然のようにコケたりします (エラーを統一とかはしてくれないので，いろんなコマンドのエラーが飛び交って地獄が融合した究極地獄をお届け)

開発者は `package.py` を用意してspackにpull requestを出すみたい(多分)

これは開発者が楽なだけで，ユーザは楽じゃないと思う．

## spackの特徴，解決したいこと
spackとは：
- LLNL(ローレンス・リバモア)が主導してるらしい
- スパコン向け．つまりターゲットユーザの想定は：
    - アーキ/コンパイラ/MPIなどのバージョンが特殊
    - 性能が欲しい
    - ユーザ権限だけでやりたい (スパコンなのでrootがない

ということで，どういうものかというと，

- インストールしたパッケージはユーザーの$HOME以下で管理
- パッケージはEnvironment Moduleの登録が行われる．moduleコマンドで切り替える
- 基本的にすべてをソースからコンパイルする
    - CUDAやMKLなど例外はある
- 最低限のコンパイラやgit, makeなどのコマンドは用意しろって要件に書いてある
    - https://spack.readthedocs.io/en/latest/getting_started.html
    - Dockerだと色々足りない．気をつけろ
    - 使うコンパイラは `$HOME/.spack/linux/compilers.yaml` に書く
- 依存関係も含めて全部コンパイルしてくれる
    - コンパイラも依存に入ってればコンパイルする．正気か？
        - 色々見たけど依存に入れてるパッケージは見当たらない
        - つまりコンパイラはお前が揃えろってこと？(まぁスパコンだし初期からコンパイラは揃ってるのか)

# spackの正体と導入
なんとgit cloneするとバイナリが降ってくる．は～～～便利な世の中になったもんじゃ～～
と思ったら中身はバイナリじゃなかった．pythonスクリプトだった: https://github.com/spack/spack/blob/develop/bin/spack
お前にはがっかりだよ

`docker run -it ubuntu:22.04` からの

```
apt insatll -y python3
git clone https://github.com/spack/spack
cd spack/bin
./spack
```

これで動く．
当然これ場所が大事で，cloneした場所に無いと依存してるpythonコードが見つからないので動かない．パスを通すなりリンクを作るなりしろということらしい．
この時点でちょっと殺意が沸くが，spack自体がアーキ依存になっては意味がないから仕方ないとも言える．結局pythonは必要なのだが．


## 全部コンパイルとは？？？正気か？？？
繰り返しますが**基本的に全部ユーザ環境でビルドする．**

ざっくりspackのコードを見てきた．
まず，ソフトウェア開発者がspackにどうやってuploadするかというと，
1. パッケージ作成スクリプトの `package.py` を作る (詳細は後述)
    - これはspack側が用意してるI/Fを使って簡単に作れるみたいな妄想らしい
    - 簡単なものなら自動生成する機構がついてるらしい
2. spackの `spack/var/repos/builtin/packages` に上記pythonコードを置く
3. packageリストのymlに自分のやつを追記する
4. spackに**プルリクエストを出す！！！！！！**
    - 本当かどうかわかんないけどPRの一覧見てるとそういうのが出てる：https://github.com/spack/spack/pull/30944

じゃあ `package.py` にどういうことを書くかというと:

1. ソースコードのtar.gz等のダウンロードURL，またはgit/svn/mercurial/goのリポジトリURL
    - URL配下にある似た名前(?)のファイルはバージョン違いとして登録される
    - ダウンロードする実体はハッシュとして管理される
        - これは破損・セキュリティ対策のチェックサムが目的
2. どのビルドツールをつかうか
    - Make, SCons, Waf, Autotools, cmake, meson, qmakeなど．．
    - List: https://spack.readthedocs.io/ja/latest/build_systems.html
3. ビルドオプションなど
4. spackの依存パッケージ

このへんである．

### ソースコードのない例：CUDAとかMKLとかどうしてんの？？
ソースのある例は後で見るとして，例外の方から見る．

コードはここ
https://github.com/spack/spack/blob/develop/var/spack/repos/builtin/packages/cuda/package.py#L27

```python=3
'11.7.0': {
        'Linux-aarch64': ('e777839a618ca9a3d5ad42ded43a1b6392af2321a7327635a4afcc986876a21b', 'https://developer.download.nvidia.com/compute/cuda/11.7.0/local_installers/cuda_11.7.0_515.43.04_linux_sbsa.run'),
        'Linux-x86_64': ('087fdfcbba1f79543b1f78e43a8dfdac5f6db242d042dde820e16dc185892f26', 'https://developer.download.nvidia.com/compute/cuda/11.7.0/local_installers/cuda_11.7.0_515.43.04_linux.run'),
        'Linux-ppc64le': ('74a507ac54067c258e6b7c9063c98d411116ecc5c5397b1f6e6a999e86dff08a', 'https://developer.download.nvidia.com/compute/cuda/11.7.0/local_installers/cuda_11.7.0_515.43.04_linux_ppc64le.run')},
'11.6.2': {
        'Linux-aarch64': ('b20c014c6bba36b13c50da167ad42e9bd1cea24f3b6297b495ea129c0889f36e', 'https://developer.download.nvidia.com/compute/cuda/11.6.2/local_installers/cuda_11.6.2_510.47.03_linux_sbsa.run'),
        'Linux-x86_64': ('99b7a73dcc52a52cef4c1fceb4a60c3015ac9b6404082c1677d9efdaba1d4593', 'https://developer.download.nvidia.com/compute/cuda/11.6.2/local_installers/cuda_11.6.2_510.47.03_linux.run'),
        'Linux-ppc64le': ('869232ff8dbf295a71609738ac9e1b0079ca75597b427f1c026f42b36896afe8', 'https://developer.download.nvidia.com/compute/cuda/11.6.2/local_installers/cuda_11.6.2_510.47.03_linux_ppc64le.run')},
```

これは．．．．ダウンロードしてるだけですね．．
確かにビルドも何もしないで，ダウンロードして解凍することだけを記述しておけばそれでいいか．．．

### ソースコードがある例：xtensorのpackage.pyを見てみる
https://github.com/spack/spack/blob/develop/var/spack/repos/builtin/packages/xtensor/package.py

見れば分かる．
ユーザはインストール時にvariantを指定したり，cmakeのargsを付け足したり出来る
当然コケたりする．俺はコケた．

```python=3
from spack.package import *

class Xtensor(CMakePackage):
    """Multi-dimensional arrays with broadcasting and lazy computing"""

    homepage = "https://github.com/xtensor-stack/xtensor-io"
    url      = "https://github.com/QuantStack/xtensor/archive/0.13.1.tar.gz"
    git      = "https://github.com/QuantStack/xtensor.git"
    maintainers = ['ax3l']

    version('develop', branch='master')
    version('0.24.1', sha256='dd1bf4c4eba5fbcf386abba2627fcb4a947d14a806c33fde82d0cc1194807ee4')
 ## バージョンがダラダラ長いので省略

    variant('xsimd', default=True,
            description='Enable SIMD intrinsics')
    variant('tbb', default=True,
            description='Enable TBB parallelization')

    depends_on('xtl', when='@develop')
    depends_on('xtl@0.7.2:0.7', when='@0.23.2:')
## 依存がいっぱい書いてあるので省略 ##
    depends_on('intel-tbb', when='+tbb')

    conflicts('%gcc@:4.8')
    conflicts('%clang@:3.5')

    def cmake_args(self):
        args = [
            self.define('BUILD_TESTS', self.run_tests),
            self.define_from_variant('XTENSOR_USE_XSIMD', 'xsimd'),
            self.define_from_variant('XTENSOR_USE_TBB', 'tbb')
        ]

        return args
```

これで `spack info xtensor` すると，こんなふうにでてくる
![](https://i.imgur.com/OWBpR6Z.png)

公式曰く，それぞれこうやって指定するんだそうな．
- `@` Optional version specifier (@1.2:1.4)
- `%` Optional compiler specifier, with an optional compiler version (gcc or gcc@4.7.3)
- `+` or `-` or `~` Optional variant specifiers (+debug, -qt, or ~qt) for boolean variants
- `name=<value>` Optional variant specifiers that are not restricted to boolean variants
- `name=<value>` Optional compiler flag specifiers. Valid flag names are cflags, cxxflags, fflags, cppflags, ldflags, and ldlibs.
- `target=<value>` `os=<value> Optional architecture specifier (target=haswell os=CNL10)
- `^` Dependency specs (^callpath@1.1)

LLVMなんかは，関連ツールやomp targetなんかを `variant` にしてる．なるほどね
https://github.com/spack/spack/blob/develop/var/spack/repos/builtin/packages/llvm/package.py#L80


# spack調べまとめ
CUDAなどの例外を除いて，大体ビルドされます．

うまくいく場合もあるけど，使われてなさそうなパッケージは多分コケます．

俺はいいや．．．解散

お役立ち日本語記事
https://www.nabe-intl.co.jp/takeruboost/spack%E3%82%92%E4%BD%BF%E3%81%A3%E3%81%9F%E5%86%8D%E7%8F%BE%E5%8F%AF%E8%83%BD%E3%81%AA%E7%92%B0%E5%A2%83%E3%81%AE%E6%A7%8B%E7%AF%89%E2%91%A0-spack-environment/

https://www.nabe-intl.co.jp/takeruboost/spack-package-manager%E3%82%92%E4%BD%BF%E3%81%A3%E3%81%A6%E6%9C%80%E6%96%B0%E3%81%AEgcc%E3%82%92%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB/