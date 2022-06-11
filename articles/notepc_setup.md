---
title: "ノートPCにUbuntu22.04を入れてサーバにするときの電源とか温度管理の設定 (メモ書き)"
type: "tech"
topics: ["Tech"]
published: true
---

# はじめに
この記事はメモ書きです．

たまにノートPCにLinuxを入れると作業手順を忘れるので

# Install
1. Ubuntuの好きなバージョンのISOを落としてくる
2. ライブメディアを作る．
    - [Rufus](https://rufus.ie/ja/)とかいうのを使っている．
3. だいたい全部何も変えずに入れる
    - パーティションを真面目に設定していた時期があったが，Dockerが `/var` を使うので， `/` と `/var` のサイズに迷う．ノートPCだとトータルのSSDの容量が大してないので，結局 `/` を大きく取ればいいやとなるのでデフォルトでいいやってなった．

# 設定 (ノートPC向け)

## sensors
温度とれるやつ．サーバにはだいたい入れてる．

取った温度はcronで上手いことやればいいと思うけど，最近真面目にやるのをやめてる．

```bash
$ sudo apt install lm-sensors 
```

`sensors` ってコマンド叩くと，こんなのがでてくる．

```bash
coretemp-isa-0000
Adapter: ISA adapter
Package id 0:  +37.0°C  (high = +100.0°C, crit = +100.0°C)
Core 0:        +37.0°C  (high = +100.0°C, crit = +100.0°C)
Core 1:        +37.0°C  (high = +100.0°C, crit = +100.0°C)
Core 2:        +36.0°C  (high = +100.0°C, crit = +100.0°C)
Core 3:        +37.0°C  (high = +100.0°C, crit = +100.0°C)

dell_smm-isa-0000
Adapter: ISA adapter
fan1:           0 RPM

nvme-pci-0200
Adapter: PCI adapter
Composite:    +25.9°C  (low  = -273.1°C, high = +82.8°C)
                       (crit = +86.8°C)
Sensor 1:     +25.9°C  (low  = -273.1°C, high = +65261.8°C)

ucsi_source_psy_USBC000:001-isa-0000
Adapter: ISA adapter
in0:          20.00 V  (min =  +5.00 V, max = +20.00 V)
curr1:         3.25 A  (max =  +3.25 A)

iwlwifi_1-virtual-0
Adapter: Virtual device
temp1:        +43.0°C

pch_cannonlake-virtual-0
Adapter: Virtual device
temp1:        +36.0°C

BAT0-acpi-0
Adapter: ACPI interface
in0:           8.44 V
curr1:       1000.00 uA
```

## powertop, tlp, tlp-rdw
[powertop](https://wiki.archlinux.jp/index.php/Powertop) は，なんか電力設定を最適化するツール
体感だけどいい感じになってる気がするので入れてる．本当のところは知らない．

tlp, tlp-rdwはpowertopをなんか適当に最適化してくれるツール．

```bash
apt install powertop tlp tlp-rdw
```

普通にpowertopに任せるならこのコマンドを使うが，再起動すると戻っちゃう

```bash
sudo powertop --auto-tune
```

tlp任せにしたかったら

```bash
sudo tlp start
```

する．これを一度やると，再起動しても自動で最適化してくれる

## (サーバ向け)蓋を閉じてもサスペンドさせない

設定ファイルからいける．
```
sudo vim /etc/systemd/logind.conf
```
して，`HandleLidSwitch=ignore` にする．ファイル変更したら再起動して．