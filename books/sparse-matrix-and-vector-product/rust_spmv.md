---
title: Rustで疎行列ベクトル積
---

# はじめに
この記事[数値計算アドベントカレンダー2022](https://qiita.com/advent-calendar/2022/numerical_analysis)の20日目の記事です．

仕事でRustを使いそうなので，勉強がてら疎行列ベクトル積 (SpMV)の性能とライブラリの充実具合を調べた内容になります．

この記事は見ての通り筆者がたまに書き進めている[C/C++で疎行列ベクトル積を行うための本](https://zenn.dev/books/sparse-matrix-and-vector-product/edit)におけるC/C++以外の言語からSpMVを行うセクションの1パートという位置づけをしています．そもそも疎行列の格納形式などについて全くわからないよという人は手前の章を参照してください．

なお，筆者は疎行列ベクトル (SpMV)には詳しいつもりですがRust初心者なのでRustについては間違いがあるかもしれません．性能の計測については[これを見ながら](https://zenn.dev/termoshtt/books/b4bce1b9ea5e6853cb07/viewer/criterion) `std::time`を使って気を使ったつもりですが，上げているコードがRustの良さが出てないクソコードであることはご容赦ください．

と，思っていたら3つのライブラリを扱う予定が1つが答えが合わず，私のせいかわからなくて困ったのでちょっとコード公開を延期します．(まぁ9割サンプルコードと同じですが．．)

ということで，この記事では答えの合うsprsとnalgebraという2つのライブラリを見ていきます．

# TL; DR
そもそもRustで書かれた疎行列ベクトル積ライブラリをもってるライブラリが2つしか見当たらず．それも全然速くなかった．というか全然並列化されてない気がする．
私がRustわかってないだけかもしれないけど並列に動いてくれない．正直コード見ても並列化されてない気がするんだけど誰か教えて．．．

# Rustという言語とHPC
Rustは近年いい感じの言語としていい感じな言語です．とりあえず安全性とかそういう話は置いといて性能の話をすると，RustはLLVMベースのコンパイル言語で静的型付けです．つまりコンパイラ最適化はC/C++と同じくらいを期待したいです．

詳しいことはtermoshtt氏の[Rustで数値計算](https://zenn.dev/termoshtt/books/b4bce1b9ea5e6853cb07)を読んで下さい．

並列化について下 (コアの中)から見ていくと，SIMDライブラリやLLVMへの自動SIMD化attributeがあるそうです．マイナーな環境では試してないけどx86なら問題なさそうです．

次のスレッドですが，OpenMPはありません．おそらくOpenMPやらOpenACC相当のコンパイラ拡張がRustに入ることはこの先無いでしょう．
スレッド並列化を行いたければ`std::thread`ライブラリ (まぁC/C++と同じような感じ)を使うか，[Rayon](https://github.com/rayon-rs/rayon)というライブラリを使います．

まぁSIMD使ってもメモリで止まるのがSpMVなので，今回はSIMDなし，Rayonを使ったスレッド並列を考えます．

え？CUDA？なにそれしらない．

# Rustの疎行列ライブラリ
## 疎行列ライブラリの選定
C/C++などのライブラリをコールするライブラリを除いてcrates.io(https://crates.io/)でSparase Matrixとか色々検索し，ヒットしたライブラリの実装状況をまとめることにします．

以下のような条件で探しました:
- 少なくとも倍精度疎行列ベクトル積があって外から利用できる (超でかいライブラリの内側に潜んでるのは知らない)
- 疎行列ベクトル積がRustで書かれている (C/C++などのラッパーではない)
- crates.ioでのダウンロード件数が1000件以上ある
- 最終更新日が2年未満になっている 
- あまりに機能が少ない/未完成/ドキュメントがないものは除く (私個人の偏見と主観に基づく)
- Sparse LUやSparse SVDなどの直接法系のものは除く (そもそもそんなにないが)

色々検索した結果，重複があるので実際の数はわかりませんが，リポジトリを見に行ったのは多分20件くらい，生き残ったのは以下のわずか3件 --> と思ったら2件でした．Cをコールするライブラリはいっぱいあるんだけどね．．．

|                                                                                        | Repository                           | version | Downloads   (All-Time)                      | 最終版のリリース日 |
|----------------------------------------------------------------------------------------|--------------------------------------|---------|---------------------------------------------|--------------------|
| [sprs](https://crates.io/crates/sprs)                                                  | https://github.com/vbarrielle/sprs   | 0.11.0  | 666,014                                     | 2021/07/23/        |
| [nalgebra](https://crates.io/crates/nalgebra-sparse) (に含まれるnalgebra-sparse crate) | https://github.com/dimforge/nalgebra | 0.31.4  | 9,656 (親Crateは5,234,170)                  | 2022/11/14/        |

sprsはその名の通りRustでSparseやるぞって感じのライブラリです．

nalgebraはDenseなども含む大型のライブラリのようで，スポンサーなどもついているようです．
中を見るといくつかのlapack関数がRustで再実装されていました．

## それぞれのライブラリの実装状況
### sprs
sprsは想定通りというか，まぁ普通の疎行列ライブラリです．
疎行列の定数倍，SpMV，SpMMなどの機能を備えます．ただ，1年半くらい更新がないのが気になるところです．

格納形式としては：
- CRS
- CSC
- Triplet Format (COOのこと)

の3つに加え，疎ベクトルと密ベクトルが実装されています．

行列クラスについて特徴的なのはviewが実装されていることかと思います ([ref](https://docs.rs/sprs/latest/sprs/struct.CsMatBase.html#constructor-methods-for-sparse-matrix-views))．

SpMVのCargo.tomlを見る限りRayonがDependenciesに含まれています．ただ，[疎行列演算が含まれているコード](https://github.com/sparsemat/sprs/blob/master/sprs/src/sparse/prod.rs)に並列化っぽい記述が見当たらない気がします．
そうは見えないけれど，forの範囲を決める `outer_iterator().enumerate()` になんか細工がしてあるんでしょうか．．．？

さらに検索するとsmmpモジュールにしかRayonが出てこないように見えます．
CRSとCRSの行列積にしか使われてないようにみえるんですが，果たして．．．

追記：死ぬほど遅いんだけどなんなん？っていう2020年のRedditの[スレ](https://www.reddit.com/r/rust/comments/gzazna/whats_the_current_state_of_rusts_sparse_linear/)を見つけた

それ以外はシンプルで，RustでSpMVを書いたらこうなると言って良いんじゃないでしょうか．
Cのシンプルコードとの比較結果に期待です．

### nalgebra
結構大きいプロジェクトnalgebraに含まれるnalgebra-sparse crateです．

格納形式としては：
- CRS
- CCS
- COO

が含まれています．疎ベクトルはないようです．

ドキュメントにこうある通りAPI策定に忙しくて並列化に手が回ってないという噂があります．
> The library is in an early, but usable state. The API has been designed to be extensible, but breaking changes will be necessary to implement several planned features. While it is backed by an extensive test suite, it has yet to be thoroughly battle-tested in real applications. Moreover, the focus so far has been on correctness and API design, with little focus on performance. Future improvements will include incremental performance enhancements.

全世界のモスの皆さん喜んでください！
このライブラリで特徴的なのは疎行列の中間レイヤの抽象化のために [serde](https://github.com/serde-rs/serde)が含まれているところです．実はこれが気になってこの記事を書き始めたと言っても過言ではありません．たとえばCRSの実装は[ここ](https://github.com/dimforge/nalgebra/blob/dev/nalgebra-sparse/src/csr/csr_serde.rs)にあります．

何を言っているのかわからない皆さんは，まずはこれを読んで意識を高めてください：https://www.ricos.co.jp/tech/serde-deserializer/

要するにシリアライズ・デシリアライズのフレームワークを使って上手いこと中間フォーマットの変換ライブラリを実装しています．この実装で変換が速いのかも調査したかったんですが，残念ながら時間が足りなかったので次回にご期待ください (次回はあるのか)．

あと誰か教えてほしいのですが，nalgebraの変換コードの部分には並列化が見つかりませんでしたが，serdeを使って変換ライブラリを書くとserdeが裏でなんかいい仕事をして並列化してくれたりするんでしょうか？

APIの策定を頑張った感出してるだけあってナウでヤングな感じのAPIが提供されています．

また，どうもSpMM(疎行列-密行列積)はありますがSpMVの関数が見当たりません．[サンプル](https://docs.rs/nalgebra-sparse/latest/nalgebra_sparse/)を見る限りすべて `spmm` としてやるようです
(確かにnx1の密行列だと言えるからそこは共通化出来るのかなるほどなあ．．．速いのかなあ．．．)．
そしてSpMMのコードがある[このへんのディレクトリ](https://github.com/dimforge/nalgebra/tree/dev/nalgebra-sparse/src/ops)を見た限り並列化はされてません．

著者がRustに詳しくないので本当かどうかはわからないのですが，どうも性能に期待できないと言うのは本当に期待できなそうです．

# 性能評価
ということで調べた段階ですでに萎え気味ではあるのですが性能を見ていきます．

MKLのコード書くの面倒くせえなあと思っていたのですが [monolish](https://github.com/ricosjp/monolish)という素敵なライブラリにMKLとSpMVとナイーブな実装のSpMV (original SpMVとよぶ)が両方あったのでこれを使うことにします．
ということで比較対象は以下の4つです．

- Rust + sprs
- Rust + nalgebra
- C + MKL (monolish MKL branch)
- C + original SpMV (monolish oss branch)

ベンチ機はIntel(R) Core(TM) i9-10900X (10C20T), 64GBでUbuntu22.04．コンパイラはCではclang14.0.5, Rustではrustc 1.65.0 (897e37553 2022-11-02), LLVM versionは15.0.0です．

対象はLaplace方程式を5点中心差分したよくある1行に5要素ある行列です．まぁベクトルと掛けるだけなんで中身は知らなくても良くて，1行に5要素が規則正しく並んでる行列だと思ってもらっていいです．

それでは結果です．単位はミリ秒，行列の行数Nを10Kから1Mまで変えてみました．

|                 | N = 10,000 | N = 100,000 | N = 1,000,000 |
|-----------------|------------|-------------|---------------|
| Rust+sprs       | 1.68       | 17.37       | 156.58    |
| Rust+nalgebra   | 1.71       | 19.11       | 174.12    |
| C+MKL           | 1.41       | 4.89        | 52.78     |
| C+original SpMV | 1.42       | 5.35        | 54.41     |

確認のために`top` くらいは見ました．プロセスのCPU使用率が100%を超えた瞬間はありませんでした．sprsについては，`sprs/benches`も動かしてみましたが，`top`の結果は同様でした．

# おわりに & 分析
ああもう嫌だ！！！

とりあえずRustわからないなりにコードは見ましたが並列化されてなさそうだし，実際動かしてみたら並列には動きませんでした．僕が間違ってる可能性は無数にあるのでおかしい所があればTwitterのDMとかで教えてください．．．

あと自分でRust+Rayonのコード書くのもやらないといけないと思って間に合わなかった．．．比較したかった．．．

あとMKL遅くね？って思う人がいるかもしれないですが，5点中心差分じゃ多分こんなもんです．ステンシルにするとかしないとMKLでも限界あると思います．
じゃあもっと面白い行列やれよとは思うんですが，正直なんか性能でなくて眠いし，5点中心差分の問題をCから吐き出してライブラリに食わせて答え合わせを書いて答え合わせを書く作業に4日もかかったのでこれ以上は今度ね．．．

正直Rustの良さを感じたかったのですが大前提として並列化されてないのでは仕方ないな感が強くて，もうちょっと調査してもこの性能なら，新しいライブラリが登場するまでは大人しくCをコールするライブラリを使ったほうが良いと思います．

まぁ次の記事はserdeを使った疎行列フォーマットの抽象化と性能か，自分でRayon使ってSpMV書いた記事になりそうな気がします．

それでは素敵なクリスマスを！！！
