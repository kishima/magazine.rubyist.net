---
layout: post
title: mruby/cで始めるオリジナルIoTデバイス作り
short_title: mruby/cで始めるオリジナルIoTデバイス作り
tags: 0059 mrubyc_iot
post_author: kishima
created_on: 2019-01-27
---
{% include base.html %}

## はじめに

こんにちは、kishimaと申します。
mruby/c（えむるびーすらっしゅしー）という言語のことを皆さんは御存知でしょうか？
この記事では、mruby/cを使ったIoTっぽい工作入門について説明していきたいと思います。

### mruby/cでIoT

筆者は仕事で馴染み深いのもあり、2017年後半あたりから電子工作を本格的に趣味で始めました。
制御にはマイコンにソフトを書き込むわけですが、言語は基本的にC言語を用いてきました。

しかしちょっとした機能の実装にいつもC言語使うのも面倒と感じる場面も多く、Rubyのようなスクリプト言語を使えたらいいなとずっと思ってきました。

最近では、mrubyという選択肢もあるわけなのですが、RAMが数百KB程度のマイコンで動かすことを考えると、mrubyは動かせないということはないですが、ある程度自由に実装するにはちょっとリソースがぎりぎりであることが多く、見送ってきました。

それでちょっと調べてみると、よりマイコン向けに特化した「mruby/c」というものが開発されていることを見つけ、これだ！と思い、早速飛びついたわけです。

本記事では、mruby/cを使って、センサからデータを取得して、モバイル通信網にデータを送ってみるための一連の流れを紹介します。本記事の内容を応用すれば、mruby/cを利用した自分の好きなIoTデバイスを作れるようになるはずです！

## mruby/cとは

mruby/cについて詳しく知らない方のために、まずmruby/cの紹介から入りたいと思います。

mruby/cとは、しまねソフト研究開発センターの[サイトの記載](https://www.s-itoc.jp/activity/research/mrubyc/)によれば、

> 「mruby/cは、Rubyの特徴を引き継ぎつつ、プログラム実行時に必要なメモリ消費量が従来のmruby（福岡で開発された組込み向けの軽量Ruby）より少ないmrubyの実装です。」

というものだそうです。

### 特徴

mruby/cの特徴をもう少し詳しく見てみましょう。

mruby/cは以下のgithubリポジトリで公開されています。

https://github.com/mrubyc/mrubyc

2019年1月現在は、Version1.2が最新となっています。
mrubyと比較して、言語としての機能がかなり制限されていますが、その分とてもコンパクトです。
githubで公開されているリポジトリのReadmeには以下のように記載されています。

|           |mruby/c | mruby |
|-----------|--------|-------|
|memory size|< 40KB  | < 400KB |
|main target|one-chip microprocessors | general embedded software |

メモリのサイズが40KB未満でも動作することが謳われており、SRAMを内蔵したワンチップマイコンのような環境でも動くことが特徴です。

### mrubyとの違い

mrubyとmruby/cは名前が似ていますが、何が違うのでしょうか？

mruby/cはmrubyのバイトコードをそのまま用いることを前提としており、mrubyのVMに相当する部分だけを提供しています。

違いを簡単に図で表してみました。

■図

さきほど、言語としての機能が制限されていると、書きましたが、一例として、mruby/cでは、mrubyと比較して、mrbgemsという機能拡張の仕組みがないため、mrubyがmrbgemsで実現している機能を実現するためには独自にポーティングする必要があります。
その他、例外やモジュールを削ったりなど、大きな違いがあります。

もう一点大きな違いとして、mruby/cではマルチタスク機能をデフォルトでサポートしている点があります。マイコンを使った開発では、外部のデバイスの監視や応答待ちなどで非同期の処理が多く発生します。mruby/cでは、そのような処理を実装するために簡易的なマルチタスク機能が利用できます。
mruby/cでは、タスク間で共通のクラス定義、グローバル変数空間を参照しており、mutexで排他処理を実装可能です。

その他には、定量的な説明が難しいポイントですが、機能を削ったことにより実装の規模が小さくなっている点があります。
実装がシンプルなので、ソースコードを眺めたときに各処理の動きの見当が付きやすく、ポーティングが非常にやりやすい点は、個人的にはとても嬉しいメリットと感じています。

## 開発環境の構築

では、本格的に開発の準備を進めていきます。

こういった電子工作や組み込み機器の開発は、PCアプリの開発と違って、ハードウェアの準備から始まるのが特徴です。
最初にモバイル通信網にデータを送ってみることを目標とすると書きましたが、特にデバイス単体で通信を行うようなデバイスで、国内で個人で手に入るものはあまり選択肢が多くありません。
（海外で売られているデバイスは日本の周波数帯と合わないか、もしくは日本での認可を得ていないことが多いので注意です）

今回はWio LTEというLTEでデータ通信可能なボードを利用したいと思います。

またボード単体では基地局と通信できないので、日本のモバイル通信網に接続可能なSIMカードも必要です。

SIMは、個人でも簡単に手に入って、初期費用も安いSORACOM Air SIMを利用します。

Wio LTEに接続するセンサのサンプルとして、たまたま手元にあったSeeedの超音波距離センサを使ってみます。

その他の準備として、PCにはArduinoIDEがインストールされており、ビルド済みのmrubyのバイナリが存在することを前提とします。

### WioLTEの説明

通信ボードの[Wio LTE JP version](https://soracom.jp/products/module/wio_lte/)について説明します。本記事では以降、Wio LTE JP versionのことを、特に断りの無い限りWio LTEと呼びます。

Wio LTEは中国深センのSeeed Studio社製のLTE通信ボードです。
制御用のマイコンがSTM32というARMのマイコンが利用されており、回路図も公開されていています。
ちゃんと認証を取得した日本用のバージョンが販売されており、日本でも安心して使えます。

単なる通信機能だけでなく、外部デバイスとの接続インタフェースとして同社製品用のGroveコネクタも搭載しています。

[SORACOMのサイト](https://soracom.jp/products/module/wio_lte/)によれば、Wio LTEのスペックは以下の通りとなっています。

|項目       | Wio LTE  |
|----------|--------------------|
|プロセッサー | STM32F405RG, ARM Cortex-M4, 168MHz |
|フラッシュメモリ | 1MB |
|内蔵SRAM | 192KB |

Wio LTEは日本向けにArduinoのライブラリが公開されていて、説明も手厚いです。
https://github.com/SeeedJP/WioLTEforArduino/wiki/Home-ja

開発環境としては、ArduinoIDEを使うのが最もお手軽かと思います。
導入については、下記の資料がわかりやすいです。
https://github.com/soracom/handson/wiki/Wio-LTE-%E3%83%8F%E3%83%B3%E3%82%BA%E3%82%AA%E3%83%B3

こちらの手順に従って、WioLTEにソフトを書き込める状態にしましょう。

**！注意！**

本記事で用いているWio LTE JP versionは、2019年1月現在、秋月電子などで在庫がありませんが、SORACOMのサイト経由で購入が可能なようです。その他にも、[Wio LTE M1/NB1(BG96)](https://soracom.jp/products/module/wio_lte_m1_nb1/)というボードもあります。そちらの利用もご検討ください。
筆者の環境ではWio LTE M1/NB1(BG96)は動作未確認ですが、本記事の内容はだいたい適用できるかと思います。

### SORACOM Air SIM for セルラーの説明

SORACOM Air SIM for セルラーは、SORACOMが販売しているデータ通信用SIMカードで、IoTのような少量のデータ通信を行うようなユースケースに適したSIMカードです。これを用いることで、SORACOMが提供するデータ管理サービスとの連携も可能になります。

詳しくは以下を参照下さい。
https://soracom.jp/services/air/

Amazonなどで購入して、公式サイトの手順に従ってWeb画面上で開通処理を行うだけですぐ使えるようになります。
通信費も安いので、個人でちょっと試したい場合にも気軽に使えると思います。

こちらもあらかじめ開通の手続きが完了していることを前提とします。

### 超音波距離センサ

Seeed社の超音波距離センサを使用します。

■図

GroveというSeeed社が提供しているインタフェースに準拠しており、Wio LTEのGroveコネクタに接続するだけで使用できて簡単です。
詳しくは下記を参照下さい。
http://wiki.seeedstudio.com/Grove-Ultrasonic_Ranger/

Groveシリーズのセンサ類は、Arduino用にドライバライブラリが公開されているのがとてもありがたいです。
こちらのUltrasonicのライブラリも公式の手順に従って、Arduinoにインストール済みであることを前提とします。

## 実装方法

ここからが本題です。

mruby/cをWio LTEの上で動かすための手順について説明していきたいと思います。

### mruby/cのポーテイング

Arduino環境にmruby/cを移植するためには以下のようなステップを踏みます。

#### mruby/cのリポジトリの取得

https://github.com/mrubyc/mrubyc からmruby/cのソースコードを取得します。
2019年1月時点ではVersion1.2が最新のリリースバージョンです。
ここではmasterの最新(2019/1/8:f864d19)を使ってみます。
（ArduinoIDEでのコンパイルエラーを避ける[bug fix](https://github.com/mrubyc/mrubyc/commit/40528070b2307ac91ef6650e0696deac160f913a)を使いたいためです）

#### Arduinoのライブラリの作成

まず、Arduinoのライブラリを作成します。
例えばMacの場合、`~/Documents/Arduino/libraries/`のようなディレクトリの下に`libmrubycForWioLTEArduino`というディレクトリを作成して、以下のような構成にします。

```
 libmrubycForWioLTEArduino/
  |- library.properties
  |- src/
```

library.propertiesは、Arduino用のライブラリの設定ファイルです。
例えば以下のような内容を書き込みます。

```
name=mruby/c for Wio LTE
version=0.0.0
author=kishima
maintainer=kishima
sentence=mruby/c implementation for Wio LTE.
paragraph=
category=Communication
url=https://github.com/kishima/libmrubycForWioLTEArduino
architectures=Seeed_STM32F4
includes=libmrubyc.h
```

このような内容を書き込むことで、ArduinoIDEのライブラリとして認識されるようになります。

#### ソースのコピー

取得したmruby/cのソースのから、src/配下のファイルを、先程作成した`libmrubycForWioLTEArduino/src`にコピーします。

mruby/cのsrcには、hal_***というディレクトリが含まれていますが、これは、各種のボードに依存している機能を切り出しものです。halは"Hardware Abstraction Layer"の略（のはず）です。
以下のように対応します。

 * hal_posix/をhal/にリネームする
 * その他のhal_***/は削除する

#### HAL(Hardware Abstraction Layer)の実装

コンソールの文字列出力がhal.hにインライン関数`hal_write`として定義されているのですが、中身は`write(1, buf, nbytes)`となっていて、そのままでは期待通り動きません。

コンソールの文字列の出力先は、ライブラリの開発者が環境に合わせて実装することが必要です。

Wio LTE Arduino Libraryでは、`SerialUSB.print`がデバッグ用シリアルへの出力を行うので、これを用いています。
注意点として、`SerialUSB.print`のようなArduinoのAPIはC++で実装されているので、hal.cからは直接呼べません。hal.cppを準備するなどして、CとC++の間を繋ぐ必要があります。

実装例を挙げると、`inline static int hal_write()`を以下のような関数に書き換えます。
（入力が2文字以上の場合、かならずヌル文字が終端にあることが前提になっています）

```C
int hal_write(int fd, const void *buf, int nbytes)
{
	char* t = (char*)buf;
	char tbuf[2];
	if(nbytes==1){
		tbuf[0]=*t;
		tbuf[1]='\0';
		hal_write_string(tbuf);
		return nbytes;
	}
	hal_write_string(t);
	return nbytes;
}
```

そして、hal.cppというファイルを作成し、`hal_write_string()`を以下のように実装します。

```C++
extern "C" void hal_write_string(char* text){
  SerialUSB.print(text);
}
```

このようにすることで、スクリプト上の`puts`の出力先がデバッグ用のシリアルになります。

もう一点、HALではタイマ割り込みの設定も考慮も必要です。mruby/cではマルチタスク機能を実現するために一定間隔でVMの処理を中断して、別のタスクのVMに処理を切り替える必要があります。
そのためにはハードウェアの周期的な割り込み機能を使うのがシンプルですが、それが使えない場合は、vm_config.hに`#define MRBC_NO_TIMER`という定義を追記すると、ソフト処理でのタスク切換えを行うようになります。

このようなポイントに注意することで、mruby/cの機能をArduinoライブラリとして利用可能となります。

実際にポーティングした結果のソースコードを下記のリポジトリにアップしていますので、詳細はそちらを参考にしてみてください。

https://github.com/kishima/libmrubycForWioLTEArduino

### C拡張の書き方

mruby/cをポーティングしただけの状態だと、シリアルのコンソールに文字が出力されるだけで、他に何もすることができません。
ボード固有の機能を利用するためにはC言語で書かれている機能をmruby/cから呼び出す必要があります。RubyのメソッドをC言語で実装するための方法を説明します。

#### クラスの追加



#### メソッドの追加


## 動かし方

自分で作ったmrubyのメソッドを利用したアプリを、実際に動かしてみましょう。

### WioLTE実機への転送

以下のようなスクリプトをtest.rbという名前で保存します。

```Ruby

Wio.init
#Wio.power_supply_LTE
#Wio.turnon_or_reset
#Wio.activate("soracom.io", "sora", "sora")

loop do
val = UltraSonic.read
puts "{distance: #{val}}"
# Wio.send("harvest.soracom.io",8514,"{distance: #{val}}")
end
```

このスクリプトを以下のようなmrubyのコマンドでバイトコードを16進数配列で表したC言語ファイルに変換します。

```
 $ mrbc -E -B code test.rb
```

test.cというファイルができるはずです、この内容をArduinoのinoファイルでincludeします。
準備ができたら、Wio LTEを接続して、DFUモードに切り替えた後、ArduinoIDEの書き込みボタンをクリックして、ビルド＆書き込みを行います。

DFUモードでは、通常のプログラムは起動せず、プログラムの書き込みを待ち受ける状態になります。
細かいボタン操作については、紹介した参考サイトを参照下さい。

### 動作チェック

では実際に動かしてみましょう。書き込んだ後にボードのリセットボタンを押すと、書き込んだプログラムが起動します。

動かしてみた結果のシリアルログを示します。
```
— begin setup
— run mruby script
class_wio_power_supply_LTE
class_wio_turnon_or_reset
TurnOnOrReset OK
send:{“distance”:158}
done
```

SORACOMの管理画面から見た結果を下記に示します。

![SORACOM Harvet画面](0059-original_mrubyc_iot_device/soracom_harvest.jpg)

実際に測定した距離の数値が転送されていることが確認できました！

この先は、Soracom Beamのような機能を利用して、AWSとの連携も可能です。後は煮るなり焼くなり自由自在というわけです。

## まとめ

この記事では、WioLTEというモバイルデータ通信が可能なボード上で、mruby/cを使って開発を行う基本的な方法を紹介しました。
環境をそろえるのが面倒だったりしますが、一度始めてしまえば意外とあっさり動くので、興味のある方はぜひトライしてみてほしいです。

## 著者について

[@kishima](https://twitter.com/kishima)

組み込みソフト系サラリーマンです。Rubyは趣味で触れ合うことが多いです。
TokyuRuby会議のスタッフをしたり、Kawasaki.rbに時々参加したりしています。
