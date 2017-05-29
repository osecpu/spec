# Opcode spec

- ここでは、OSECPUのCPUコードを定義する。
- ベースとなるバックエンド命令セット: http://osecpu.osask.jp/wiki/?page0072
  - ここから、ソフトウエア的にCPUコードに変換して実行する。
  - 変換プログラムはCPUコードで記述し、ROMに格納する。
  - 変換時の留意点については [こちら](./back2cpu.md)を参照。
- 命令フォーマットの詳細については、以下のスプレッドシートに記述している。
  - https://docs.google.com/spreadsheets/d/1Mu7-vil4o_n7MA1vi8nemQ8nVOJB7aya1bV88G011OA/edit?usp=sharing

## 発生する可能性のある例外
- CPUは以下の例外を発生させる可能性がある。
  - 例外が発生した場合、CPUの動作は停止され、例外ポートに例外コードが出力される。
  - 将来的には、例外ハンドラ機能を用意するかもしれないが、現在は存在しない。
  
- #UD: 無効命令例外
- #SE: セキュリティ例外
- #DE: 除算例外
- (#OE: オーバーフロー例外)

## 実行環境
- 32bit符号付き整数レジスタx64(R0-R3F)

- ポインタレジスタx64(P0-P3F)
  - 1エントリの構造(28bit)
 
|bit  |0 - 11     |12 - 27   |
|----:|:---------:|:--------:|
|field|LBID(12bit)|ofs(16bit)|

- ラベルテーブルx4096(LBT)
  - 1エントリの構造(40bit)
  
|bit  |0 - 7    |8 - 23     |24-39     |
|----:|:-------:|:---------:|:--------:|
|field|typ(8bit)|base(16bit)|count(16bit)|

## 定数等
- メモリtypに指定可能な値は以下のとおりである。
  - バックエンド命令におけるtypフィールドのサブセット+Codeになっている。

|typ(Hex)|データ型   |
|--------|---------|
|0x00    |Undefined|
|0x01    |VPtr     |
|0x02    |SINT8    |
|0x03    |UINT8    |
|0x04    |SINT16   |
|0x05    |UINT16   |
|0x06    |SINT32   |
|0x07    |(UINT32) |
|0x08    |SINT4    |
|0x09    |UINT4    |
|0x0A    |SINT2    |
|0x0B    |UINT2    |
|0x0C    |SINT1    |
|0x0D    |UINT1    |
|0x86    |Code     |

- 上記以外の値がtypに指定された場合は無効命令例外(#UD)が発生する。
- 32bit OSECPUでは、32ビット符号なし整数は指定できない。

## 形式的記述の例
```
// 整数レジスタ
R[0x3F] <- 0              // R3Fに0を代入

// ポインタレジスタ
P[0x3F].LBID              // P3FのLBID
P[0x3F].ofs               // P3Fのofs

// ラベルテーブル
LBT[0].typ                // ラベル0のtyp
LBT[0].base               // ラベル0のbase
LBT[0].count              // ラベル0のcount

// ポインタレジスタとラベルテーブルの組み合わせ
LBT[P[0x3F].LBID].typ	  // P3Fが指すラベルのtyp

// 命令レジスタと命令カウンタ（内部）
// 命令レジスタIRはIR[0]とIR[1]の2個ある。
IR[0] = MEM[PC]		      // プログラムカウンタの指す命令をIR0にフェッチ
PC <- PC + 1		      // プログラムカウンタをインクリメント
```

## 01: LBSET
### オペランド
- typ: データタイプ
- LBID: ラベル番号
- base: 符号なし整数
- count: 符号なし整数

### 動作
- メモリのbase番地を開始点として、typのデータがcount個存在するようなラベルを定義する。

```
IR[0] <- MEM[PC]
PC++
IR[1] <- MEM[PC]
PC++
LBT[IR[0].LBID].typ = IR[0].typ
LBT[IR[0].LBID].base = IR[0].base
LBT[IR[0].LBID].count = IR[0].count
```

## 02: LIMM16
### オペランド
- r: 整数レジスタ番号
- imm16: 16bit符号あり整数

### 動作
- r = imm16
- imm16は符号拡張されてから代入される。

```
IR[0] <- MEM[PC]
PC++
R[IR[0].r] = IR[0].imm16
```

## 03: PLIMM
### オペランド
- p: ポインタレジスタ番号
- LBID: 符号なし16bit整数

### 動作
- ポインタレジスタpにラベル番号LBIDを代入する。ポインタのオフセットは0に初期化される。

- pがP3Fの場合はJMP命令となる。このとき、代入されるラベルのtypはCodeでなければならない。そうでない場合、セキュリティ例外が発生する。

```
IR[0] <- MEM[PC]
PC++
if(IR[0].p == 0x3F){
	// JMP
} else{
	// Load data label
	if(LBT[IR[0].LBID].typ is not data) raise(#SE);
	P[IR[0].p].typ = LBT[IR[0].LBID].typ
	P[IR[0].p].base = LBT[IR[0].LBID].base
	P[IR[0].p].count = LBT[IR[0].LBID].count
}
```


## 04: CND
### オペランド
- r: 整数レジスタ番号
### 動作
- 後続する1命令の実行を整数レジスタrの内容に応じてスキップする。
  - レジスタrのLSBが1: 後続する1命令を実行する。
  - レジスタrのLSBが0: 後続する1命令は実行しない。

```
IR[0] <- MEM[PC]
PC++
if(R[IR[0].r} & 0x01 == 0){
	// skip next instruction
	PC++
}
```

## 08: LMEM
### オペランド
- r: 整数レジスタ番号
- p: ポインタレジスタ番号
- typ: データタイプ
### 動作
- ポインタレジスタpが指す内容を整数レジスタrに代入する。
- ポインタレジスタpが指すラベルのデータ型はtypと一致しなければならない。
  - 一致しない場合はセキュリティ例外(#SE)が発生する。
- ポインタレジスタpのオフセットは、そのポインタの指すラベルのcount未満でなければならない。
  - そうでない場合はセキュリティ例外(#SE)が発生する。
- この命令は、32bit幅に展開済みのデータを常に読み込む。
  - パックされた状態のデータを読み込みたい場合には、`LMEMCONV`命令を使用すること。

```
IR[0] <- MEM[PC]
PC++

if(LBT[P[IR[0].p].LBID].typ == Undefined) raise(#SE)
if(LBT[P[IR[0].p].LBID].typ != IR[0].typ) raise(#SE)
if(LBT[P[IR[0].p].LBID].typ is not data)  raise(#SE)
if(P[IR[0].p].ofs >= LBT[P[IR[0].p].LBID].count) raise(#SE)
R[IR[0].r] = MEM[LBT[P[IR[0].p].LBID].base + P[IR[0].p].ofs]

```

## 09: SMEM
### オペランド
- r: 整数レジスタ番号
- p: ポインタレジスタ番号
- typ: データタイプ
### 動作
- 整数レジスタrの内容をポインタレジスタpが指す位置に書き込む。
- ポインタレジスタpが指すラベルのデータ型はtypと一致しなければならない。
  - 一致しない場合はセキュリティ例外(#SE)が発生する。
- この命令では、32bit幅に値が展開される（レジスタのデータがそのまま書き込まれる）。
  - 現段階で、パックされた状態のメモリに対して演算を行う命令は用意されていない。

```
IR[0] <- MEM[PC]
PC++

if(LBT[P[IR[0].p].LBID].typ == Undefined) raise(#SE)
if(LBT[P[IR[0].p].LBID].typ != IR[0].typ) raise(#SE)
if(LBT[P[IR[0].p].LBID].typ is not data)  raise(#SE)
if(P[IR[0].p].ofs >= LBT[P[IR[0].p].LBID].count) raise(#SE)
MEM[LBT[P[IR[0].p].LBID].base + P[IR[0].p].ofs] = R[IR[0].r]
```


## 0A: PLMEM
### オペランド
- p0: ポインタレジスタ番号
- p1: ポインタレジスタ番号
- typ: データタイプ

### 動作
- p0が指すポインタ情報をp1に代入する。
- typはVPtrでなければならない。
  - そうでなければセキュリティ例外が発生する。

```
IR[0] <- MEM[PC]
PC++

if(LBT[P[IR[0].p0].LBID].typ != VPtr) raise(#SE)
if(P[IR[0].p0].ofs >= LBT[P[IR[0].p0].LBID].count) raise(#SE)
P[IR[0].p1] = MEM[LBT[P[IR[0].p0].LBID].base + P[IR[0].p0].ofs]
// TODO: Fix bit fields for VPtr
```


## 0B: PSMEM
### オペランド
- p0: ポインタレジスタ番号
- p1: ポインタレジスタ番号
- typ: データタイプ

### 動作
- ポインタp1に相当する情報をp0が指すメモリ領域に書き込む。
  - 書き込まれる情報には、ポインタのオフセットも含まれる。
- typはVPtrでなければならない。
  - そうでなければセキュリティ例外が発生する。

```
IR[0] <- MEM[PC]
PC++

if(LBT[P[IR[0].p0].LBID].typ != VPtr) raise(#SE)
if(P[IR[0].p0].ofs >= LBT[P[IR[0].p0].LBID].count) raise(#SE)
MEM[LBT[P[IR[0].p0].LBID].base + P[IR[0].p0].ofs] = P[IR[0].p1]
// TODO: Fix bit fields for VPtr
```

## 0E: PADD
### オペランド
- r: 整数レジスタ番号
- p0: ポインタレジスタ番号
- p1: ポインタレジスタ番号
- typ: データタイプ

### 動作
- p0 = p1 + r 
- p1のオフセットにrを足したものをp0に代入する。
- typはp1の指すtypと一致していなければならない。
  - そうでなければセキュリティ例外が発生する。
- この命令によりポインタが移動した結果、オフセットがラベルの範囲外になっても、例外は発生しない。

```
IR[0] <- MEM[PC]
PC++

if(LBT[P[IR[0].p1].LBID].typ != IR[0].typ) raise(#SE)
P[IR[0].p0].LBID = P[IR[0].p1].LBID
P[IR[0].p0].ofs = P[IR[0].p1].ofs + R[IR[0].r]
```

## 0F: PDIF
### オペランド
- r: 整数レジスタ番号
- p0: ポインタレジスタ番号
- p1: ポインタレジスタ番号
- typ: データタイプ
### 動作
- r = p0 - p1 
- p0のオフセットからp1のオフセットを引いた結果を整数レジスタrに代入する。
- typはp1およびp0のtypと一致していなければならない。
  - そうでなければセキュリティ例外が発生する。
- p1およびp0のLBIDはお互いに一致していなければならない。
  - そうでなければセキュリティ例外が発生する。

```
IR[0] <- MEM[PC]
PC++

if(LBT[P[IR[0].p0].LBID].typ != IR[0].typ) raise(#SE)
if(LBT[P[IR[0].p1].LBID].typ != IR[0].typ) raise(#SE)
if(P[IR[0].p1].LBID != P[IR[0].p1].LBID) raise(#SE)
P[IR[0].p0].LBID = P[IR[0].p1].LBID
P[IR[0].p0].ofs = P[IR[0].p1].ofs + R[IR[0].r]
```

## 10-12: OR, XOR, AND
### オペランド
- r0, r1, r2: 整数レジスタ番号
### 動作
- r0 = r1 | r2
- r0 = r1 ^ r2
- r0 = r1 & r2
- ビットごとの論理演算を行う。

```
IR[0] <- MEM[PC]
PC++

if(IR[0] == 0x10) R[IR[0].r0] = R[IR[0].r1] | R[IR[0].r2]
if(IR[0] == 0x11) R[IR[0].r0] = R[IR[0].r1] ^ R[IR[0].r2]
if(IR[0] == 0x12) R[IR[0].r0] = R[IR[0].r1] & R[IR[0].r2]
```

## 14-15: ADD, SUB
### オペランド
- r0, r1, r2: 整数レジスタ番号
### 動作
- r0 = r1 + r2
- r0 = r1 - r2
- 符号付き整数として演算を行う。
- オーバーフロー/アンダーフローが発生した場合はセキュリティ例外が発生する。

```
IR[0] <- MEM[PC]
PC++

if(IR[0] == 0x14) R[IR[0].r0] = R[IR[0].r1] + R[IR[0].r2]
if(IR[0] == 0x15) R[IR[0].r0] = R[IR[0].r1] - R[IR[0].r2]
if(ALU.OF) raise (#SE)
```

## 16: MUL
### オペランド
- r0, r1, r2: 整数レジスタ番号
### 動作
- r0 = r1 * r2
- オーバーフローが発生した場合はセキュリティ例外が発生する。

```
IR[0] <- MEM[PC]
PC++

if(IR[0] == 0x16) R[IR[0].r0] = R[IR[0].r1] * R[IR[0].r2]
if(ALU.OF) raise (#SE)
```

## 18-19: SHL, SAR
### オペランド
- r0, r1, r2: 整数レジスタ番号
### 動作
- r0 = r1 << r2
- r0 = r1 >> r2
- 右シフトは算術シフトで行われます。
```
IR[0] <- MEM[PC]
PC++

if(IR[0] == 0x18) R[IR[0].r0] = R[IR[0].r1] << R[IR[0].r2]
if(IR[0] == 0x19) R[IR[0].r0] = R[IR[0].r1] >> R[IR[0].r2]
```

## 1A-1B: DIV, MOD
### オペランド
- r0, r1, r2: 整数レジスタ番号
### 動作
- r0 = r1 / r2
- r0 = r1 % r2
- r2が0の場合は除算例外(#DE)が発生する。

```
IR[0] <- MEM[PC]
PC++

if(R[IR[0].r2] == 0) raise(#DE)

if(IR[0] == 0x18) R[IR[0].r0] = R[IR[0].r1] / R[IR[0].r2]
if(IR[0] == 0x19) R[IR[0].r0] = R[IR[0].r1] % R[IR[0].r2]
```

## 1E: PCP
### オペランド
- p0, p1: ポインタレジスタ番号
### 動作
- p0 = p1
- p0のポインタの指す先を、p1のポインタと等しくする。
```
IR[0] <- MEM[PC]
PC++

P[IR[0].p0].LBID = P[IR[0].p1].LBID
P[IR[0].p0].ofs = P[IR[0].p1].ofs
```

## 20-25: CMP(E|NE|L|GE|LE|G)
### オペランド
- r0, r1, r2: 整数レジスタ番号
### 動作
- r0 = (r1 == r2)
- r0 = (r1 != r2)
- r0 = (r1 < r2)
- r0 = (r1 >= r2)
- r0 = (r1 <= r2)
- r0 = (r1 > r2)

- r0には右辺の真偽値(真ならば1, 偽ならば0)が代入される。

```
IR[0] <- MEM[PC]
PC++

R[IR[0].r1] - R[IR[0].r2]

if(IR[0] == 0x20) R[IR[0].r0] = ALU.ZF
if(IR[0] == 0x21) R[IR[0].r0] = !ALU.ZF
if(IR[0] == 0x22) R[IR[0].r0] = ALU.NF
if(IR[0] == 0x23) R[IR[0].r0] = !ALU.NF
if(IR[0] == 0x24) R[IR[0].r0] = ALU.NF | ALU.ZF
if(IR[0] == 0x25) R[IR[0].r0] = !ALU.NF & !ALU.ZF
```

## 26-27: TST(Z|NZ)
### オペランド
- r0, r1, r2: 整数レジスタ番号
### 動作
- r0 = ((r1 & r2) == 0)
- r0 = ((r1 & r2) != 0)
- r0には右辺の真偽値(真ならば1, 偽ならば0)が代入される。

```
IR[0] <- MEM[PC]
PC++

R[IR[0].r1] & R[IR[0].r2]

if(IR[0] == 0x26) R[IR[0].r0] = ALU.ZF
if(IR[0] == 0x27) R[IR[0].r0] = !ALU.ZF
```

## 28-2D: PCMP(E|NE|L|GE|LE|G)
### オペランド
- r0: 整数レジスタ番号
- p1, p2: ポインタレジスタ番号
### 動作
- r0 = (p1.ofs == p2.ofs)
- r0 = (p1.ofs != p2.ofs)
- r0 = (p1.ofs <  p2.ofs)
- r0 = (p1.ofs >= p2.ofs)
- r0 = (p1.ofs <= p2.ofs)
- r0 = (p1.ofs >  p2.ofs)
- p1とp2が同一のラベルを参照していない場合はセキュリティ例外。
- r0には右辺の真偽値(真ならば1, 偽ならば0)が代入される。

```
IR[0] <- MEM[PC]
PC++

if(P[IR[0].p1].LBID != P[IR[0].p2].LBID) raise(#SE)

P[IR[0].p1].ofs - P[IR[0].p2].LBID

if(IR[0] == 0x28) R[IR[0].r0] = ALU.ZF
if(IR[0] == 0x29) R[IR[0].r0] = !ALU.ZF
if(IR[0] == 0x2A) R[IR[0].r0] = ALU.NF
if(IR[0] == 0x2B) R[IR[0].r0] = !ALU.NF
if(IR[0] == 0x2C) R[IR[0].r0] = ALU.NF | ALU.ZF
if(IR[0] == 0x2D) R[IR[0].r0] = !ALU.NF & !ALU.ZF
```

## D0: LIMM32
### オペランド
- r: 整数レジスタ番号
- imm32: 32ビット符号付き即値

### 動作
- r = imm32

```
IR[0] <- MEM[PC]
PC++

IR[1] <- MEM[PC]
PC++

R[IR[0].r] = IR[1]
```

## D1: LMEMCONV
### オペランド
- r: 整数レジスタ番号
- p: ポインタレジスタ番号
- typ: データタイプ
### 動作
- ポインタレジスタpが指す内容を整数レジスタrに代入する。
- ポインタレジスタpが指すラベルのデータ型はtypと一致しなければならない。
  - 一致しない場合はセキュリティ例外(#SE)が発生する。
- ポインタレジスタpのオフセットは、そのポインタの指すラベルのcount未満でなければならない。
  - そうでない場合はセキュリティ例外(#SE)が発生する。
- この命令は、パックされた状態のデータを読み込み、符号拡張して整数レジスタに代入する。

```
IR[0] <- MEM[PC]
PC++

if(LBT[P[IR[0].p].LBID].typ == Undefined) raise(#SE)
if(LBT[P[IR[0].p].LBID].typ != IR[0].typ) raise(#SE)
if(LBT[P[IR[0].p].LBID].typ is not data)  raise(#SE)
if(P[IR[0].p].ofs >= LBT[P[IR[0].p].LBID].count) raise(#SE)

TMP_ADDR = (LBT[P[IR[0].p].LBID].base + P[IR[0].p].ofs)

switch(IR[0].typ >> 1){
	case 0b000:
	case 0b011:
		// 32bit
		TMP_DATA = MEM[TMP_ADDR]
		break;
	case 0b010:
		// 16bit
		TMP_ADDR = TMP_ADDR >> 1
		TMP_DATA = MEM[TMP_ADDR] >> (2 - 1 - (TMP_ADDR & 0b1)) & 0xffff
		if(IR[0].typ & 1) TMP_DATA = sigext(TMP_DATA)
		break;
	case 0b010:
		// 8bit
		TMP_ADDR = TMP_ADDR >> 2
		TMP_DATA = MEM[TMP_ADDR] >> (4 - 1 - (TMP_ADDR & 0b11)) & 0xff
		if(IR[0].typ & 1) TMP_DATA = sigext(TMP_DATA)
		break;
	case 0b010:
		// 4bit
		TMP_ADDR = TMP_ADDR >> 3
		TMP_DATA = MEM[TMP_ADDR] >> (8 - 1 - (TMP_ADDR & 0b111)) & 0xf
		if(IR[0].typ & 1) TMP_DATA = sigext(TMP_DATA)
		break;
	case 0b010:
		// 2bit
		TMP_ADDR = TMP_ADDR >> 4
		TMP_DATA = MEM[TMP_ADDR] >> (16 - 1 - (TMP_ADDR & 0b1111)) & 0b11
		if(IR[0].typ & 1) TMP_DATA = sigext(TMP_DATA)
		break;
	case 0b010:
		// 1bit
		TMP_ADDR = TMP_ADDR >> 5
		TMP_DATA = MEM[TMP_ADDR] >> (32 - 1 - (TMP_ADDR & 0b11111)) & 0b1
		if(IR[0].typ & 1) TMP_DATA = sigext(TMP_DATA)
		break;
}

R[IR[0].r] <- TMP_DATA

```

## F0: END
### オペランド
なし

### 動作
- CPUの動作を停止させる。

```
CR.HLT <- 1
```
