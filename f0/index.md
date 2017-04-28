# OSECPU rev.f0 Specification
- rev.f0は、FPGA上で直接実装することを念頭において設計された命令セットです。

## 何を大事にするのか
- OSECPUは「セキュリティを重視した命令セット」と「バイトコード密度」が主要な特徴であるから、これを直接サポートしたCPUを実装する。
- この命令セットは4ビット単位なので、4ビット単位のメモリアクセスをハードウエアでサポートしたい。


### セキュリティを重視した命令セットとはなにか
- ラベル位置にしか飛べない。データはもちろん実行できない。
- メモリアクセス時の範囲チェックおよびタイプチェックを強制する。

### バイトコード密度
- バイトコードには複数のレベルがある。
- CPUで直接可変長(hh4)エンコードの命令を実行するのは効率が悪い。
- そこで、CPUの命令セットとしては固定長のものを新たに設計する。
  - ただし、CPUに入力する命令セットとしては、可変長のバックエンドコード（すでに設計されているもの）を採用する。

### 参考になるページ
- [OSECPUが検出できる脆弱性](http://osecpu.osask.jp/wiki/?page0005)
- [OSECPUでのポインタの扱い](http://osecpu.osask.jp/wiki/?page0006)
- [OSECPUでの関数呼び出し](http://osecpu.osask.jp/wiki/?page0007)
- [OSECPUの基本設計思想](http://osecpu.osask.jp/wiki/?page0009)
- [エラーの話題](http://osecpu.osask.jp/wiki/?page0010)
- [OSECPUのQ&A](http://osecpu.osask.jp/wiki/?page0011)
- [OSECPUはどのくらいセキュアか](http://osecpu.osask.jp/wiki/?page0018)
- [アクセス権問題について](http://osecpu.osask.jp/wiki/?page0020)
- [高速化のアイデア](http://osecpu.osask.jp/wiki/?page0022)
- [OSECPUの歴史](http://osecpu.osask.jp/wiki/?page0028)
- [メモリアクセスの高速化](http://osecpu.osask.jp/wiki/?page0029)
- [OSECPU-ASKAの注意点](http://osecpu.osask.jp/wiki/?page0030)
- [フロントエンドバイトコード #0](http://osecpu.osask.jp/wiki/?page0031)

- [Rev2のバックエンド命令セット（バイト単位版）](http://osecpu.osask.jp/wiki/?page0090)

## 実装する機能
- ラベルのみにジャンプできる

## 実装しないがOSECPU-VMで特徴的な機能
- [整数レジスタのbit属性](http://osecpu.osask.jp/wiki/?page0078)

## 実行環境
- 64本の32bit符号付き整数レジスタ
- 64本のポインタレジスタ
- xxBytesのメインメモリ（データ・プログラム共用）
  - アドレス幅は32bit（命令コード上は関係ないが。）
- xxBytesのRAM（ファームウエア用）

- 256エントリ分のラベルテーブル（これも命令コード上は関係ないが。） 
  - ラベル番号は0-255(1Byte)

## 起動シーケンス
電源を入れたらRAMの変換プログラム（位置と範囲は予め設定しておく）をロードする。



