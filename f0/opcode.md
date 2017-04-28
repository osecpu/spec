# Opcode spec

- ここでは、OSECPUのCPUコードを定義する。
- ベースとなるバックエンド命令セット: http://osecpu.osask.jp/wiki/?page0072
  - ここから、ソフトウエア的にCPUコードに変換して実行する。
  - 変換プログラムはCPUコードで記述し、ROMに格納する。

## 01: LBSET
### オペランド
- typ: 符号なし整数
- base: 符号なし整数
- count: 符号なし整数

### 動作
- メモリのbase番地を開始点として、typのデータがcount個存在するようなラベルを定義する。

- typに指定可能な値は以下のとおりである。

|typ(Hex)|データ型 |
|--------|---------|
|0x00    |Undefined|
|0x01    |VPtr     |
|0x02    |SINT8    |
|0x03    |UINT8    |
|0x04    |SINT16   |
|0x05    |UINT16   |
|0x06    |SINT32   |
|0x07    |UINT32   |
|0x08    |SINT4    |
|0x09    |UINT4    |
|0x0A    |SINT2    |
|0x0B    |UINT2    |
|0x0C    |SINT1    |
|0x0D    |UINT1    |
|0x3F    |Code     |

