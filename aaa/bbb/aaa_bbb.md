## 目次

[[_TOC_]]

# 主機能
* IQデータ処理
  * AAA部Data Buffer Read制御
  * RB単位のビット幅調整(指数化)
* U-plane packet構築
  * eCPRI Common Header付加
  * Application Layer Header付加
  * IQ Dataマッピング
* 速度変換(245.76MHz→390.625MHz)
# DU冶具差分
* Ant/Car数変更
  * 8Ant/2Cr → 4Ant/2Cr
* U-Planeパケット生成時のCUPORTID付加方法変更
  * FC-Plane情報からの値 → レジスタ設定値
* 異常処理及び監視系機能追加
  * Buffer OverFlow時のBuffer内のDataクリア処理追加

# 機能概要
## ブロック図
![BBB_PGEN_UP_block](uploads/005a21d495f7b118a995753a274993bb/BBB_PGEN_UP_block.png)
<!--{{ref_image BBB_PGEN_UP_block.png,,90%}}-->

# 機能詳細
## IQデータ処理
### U-planeパケット分割判定
<!--//* FC-planeで受信するStartPRBとNumberPRBはU-planeパケットサイズを考慮したものではない-->
* TCBからのsymbolタイミング情報（CP Remove位置による遅延調整後)を基準に以下の処理を行う
  * 該当symbolのFC-plane情報をRead
  * 1パケット内に格納可能なRB数か判定
  * 1パケット内に格納できない場合は、パケット生成毎にMaxRB数を減算し1パケット内に格納可能となるまで繰り返す
  * 最終パケット生成後、次のFC-plane情報をRead
<!--* <font color="Red">IQは14bit</font>とし、1パケット内に格納可能なRB数の関係は以下の通り-->
* **IQは14bit**とし、1パケット内に格納可能なRB数の関係は以下の通り

| bit数 | Header分 | 1RB Bit数 | MaxRB数 |  
| ----- | -------- | --------- | ------- |
| 14    | 96       | 344       | 34     |

* **NumberPRB=0の場合、StartPRB=0、NumberPRB=MaxRB(Reg設定)として動作する**  
```
,{{tcopy 番地名_機能名,LS2_RegisterMap_BBB,5,[1:16]}}
,{{tcopy BW100MPRM_RBNUM,LS2_RegisterMap_BBB,5,[1:16]}}
```
tcopyどうしよう。  

|アドレス|反復|反復数|Blk|番地名_機能名|機能|MSB|LSB|初期値|説明 (ver.1.3; last 2019/3/15)|iBUS R|iBUS W|Logic Get|Logic GetEn|Logic Put|Logic PutOpt|
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--| 
|A220003C | | |BBB|BW100MPRM_RBNUM|Max RB Number|24|16|01100000|最終RB番号 設定範囲:0-272|Rd|Wr|valLv|Z|Z|Z| 

# Data Buffer Read制御
<!--//* U-planeデータはFC-planeの情報を元にパケットを生成する-->
* StartPRBから順にRB単位(12SC)でAAA部からDataをReadする
* 各I/Qデータは仮数部16bit + 指数部4bit(IQ共通)の構成
* 1RB分のデータ処理(指数共通化)が完了したら次のRBデータをReadする
* RB='1'の場合は、1つおきのRBを読み出す。(RB数はRBの数が指定される)
* AAA-BBBインタフェースは以下
![AAA-BBBインタフェース_sub6_](uploads/3feb8485c373d26beda1b3e4c8f912bd/AAA-BBBインタフェース_sub6_.png)
<!--{{ref_image AAA-BBBインタフェース(sub6).png,,90%}}-->

# RB単位のビット幅調整(指数化)
* U-planeデータはRB毎に共通の指数部を持つデータとして構成される
* RB毎にAAA部から1RB(12SC)分データをReadし、指数部及びビット幅を合わせたデータを生成する

* 指数化は以下の順で行う
```
 1. Max値の算出
  1-1. 絶対値化
  1-2. 指数部の値が最も大きいものを選択
  1-3. 指数部の値が同じ場合は、絶対値の大きい値を選択
 
 2. Max値の補正値算出
  2-1. MSBと異なるビットが出現するビット位置判定(=補正値)
  2-2. 補正値がMax値の指数以上の場合は、Max値の指数を上限とする
 
 3. Roundによる桁上げ判定
  3-1. Max値を補正値分左シフト
  3-2. BitW毎の最大値とシフト後のMax値を比較
  3-3. BitWよりもMax値の有効桁数が大きい場合は、指数の補正値分も加算
       (BitW=14の場合、指数の補正は+1～+3)
       (Max値 >= 0xFFFEの場合、指数の補正分が+2、丸めの補正分が+1の合計+3)
 
 4. 各SCデータに対する指数化
  4-1. 2項で求めた補正値分左シフト
  4-2. 4-1の結果をLSB側に0を付加して32bit化
  4-3. Max値の指数とSCデータの指数の差分を求め、右シフト
  4-4. 32bit→14bitにRound/Clip
```
* 指数=15のときに丸めで桁上げが発生した場合は、桁上げオーバフローとしてレジスタ通知を行う。(指数は15で出力される)  
```
,{{tcopy 番地名_機能名,LS2_RegisterMap_BBB,5,[1:16]}}
,{{tcopy CAROVF_*,LS2_RegisterMap_BBB,5,[1:16]}}
```  
|アドレス|反復|反復数|Blk|番地名_機能名|機能|MSB|LSB|初期値|説明 (ver.1.3; last 2019/3/15)|iBUS R|iBUS W|Logic Get|Logic GetEn|Logic Put|Logic PutOpt|
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--| 
|A2200904 | | |BBB |CAROVF_* |U-plane指数化時の桁上げオーバーフロー |7 |0 |00000000 |指数=15でbitWによる丸めで桁上げ発生 |rdClr |Z |Z |Z |Put |enBit |

