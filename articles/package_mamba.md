---
title: "パッケージ配布について考える1：conda/mamba"
type: "tech"
topics: ["Tech", "C++"]
published: true
---

# はじめに
パッケージマネージャmambaとmambaプロジェクトについて調べる．

mamba Link: https://github.com/mamba-org/mamba

# mambaが解決したいこと
conda / conda forge依存のソフトウェアが多い中で，これらは

- パッケージのサーチやダウンロードが遅い
- condaリポジトリが自由に建てられない
    - conda-forgeのクローンサーバが建てたい
- conda-buildが遅い
    - conda-build = condaパッケージ作るやつ

を解決したいらしい

# mamba調べまとめ
- 性能以外，Anacodaと変わんない
    - 普通に仮想環境作って使う
    - 当然，conda-forgeにないものはなく，より良い/新しいものを見つけてきたりはしないっぽい
    - クエリの指定構文がちょっと便利になってるらしい
- パッケージ管理の方法やディレクトリはAnacondaと同じっぽい(?)
    - 完全に同じかはしらないが ~/miniconda3/` に落ちてきた
    - ちゃんとリストも共通化されてる．condaで落とせばmambaでもalready installed
- 確かにconda-forgeから探したりするの速い
    - パッケージのダウンロードやconda-forgeの探索が並列化されてるらしい
    - 1.5倍くらい速い
- conda-buildの代替のboaも速いらしいけど調べてない

# mambaプロジェクト
5つの大きいプロジェクトが走ってるっぽい
1. mamba/micromamba
    - Anaconda/minicondaとだいたい同じ
    - mambaはcondaコマンドでインストールできる (今回こっち)
    - microcondaはwgetでバイナリ落としてくれば使える
2. Quetz
    - オープンソースのconda forge
    - dockerとかで建てられる
    - プロジェクト内部のパッケージ管理などに是非？
    - チャンネルの指定は `mamba install -c http://localhost:8000/get/channel0 libgoma` とかやる
3. Boa
    - boa is still a work-in-progress.
    - conda-buildの代替
    - 速いらしい．試してない
4. Mamba Navigator
    - Webベースのconda依存関係ビューア？
5. setup-mamba
    - This action is still under development and has not yet been published.
    - github actionsのActionになる．．．っぽい

会社にQuetzサーバ立てると便利そうではある．

# 導入
`docker run -it ubuntu:22.04` からの

mambaなら

```bash
apt update -y; apt install -y wget

# install miniconda
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash ./Miniconda3-latest-Linux-x86_64.sh #なんか色々聞かれるので頑張って答える

# ここでシェル再起動 #

conda install mamba -n base -c conda-forge -y
```

とやるか，micromambaなら，

```
apt update -y; apt install bzip2 curl
curl -Ls https://micro.mamba.pm/api/micromamba/linux-64/latest | tar -xvj bin/micromamba
./bin/micromamba shell init -s bash -p ~/micromamba
source ~/.bashrc
```

とやったら動くと書いてあるが，PATHが見つからないとかいっぱいエラーが出て死んだ．
眠いから諦めた

# 性能
なんか依存の多そうなやつをやってみる．
インストールするパッケージは完全に同じだった．
パッケージが共通化されてるので，コンテナそのものを変えて計測してる

```
time conda install -c conda-forge pytorch -y
```

- conda: 24.7 sec
- mamba: 16.3 sec

ちゃんと速かった．

# ぼやき
速いのは速いけど，Anacondaとやれることは同じだった．
conda forgeにないものは結局使えんしなあ．

Anaconda自体の話だが，MKLやCUDAがあるしOS依存じゃないから便利っちゃ便利
`libomptarget` が含まれる `libomp-dev` とかはconda forgeになかった．

OS依存の解決，バージョン管理，配布という点で当然便利なんだけど，ユーザにcondaの仮想環境を強いるのがなあ．．