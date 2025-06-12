# MIPS命令セット学習ガイド

## MIPSの基本概念

### オペランド
- **32本のレジスタ**: `$s0-$s7`, `$t0-$t9`, `$zero`, `$a0-$a3`, `$v0-$v1`, `$ra`, `$sp`など
- **2³⁰のメモリ語**: バイト・アドレス可能、4バイト境界でアクセス
- **定数**: 16ビット定数（-2¹⁵ ～ +2¹⁵-1）

### レジスタの役割
| レジスタ | 用途 | 備考 |
|----------|------|------|
| `$zero` | 常に0 | 読み取り専用 |
| `$s0-$s7` | 保存レジスタ | 関数呼び出し間で値を保持 |
| `$t0-$t9` | 一時レジスタ | 一時的な計算用 |
| `$a0-$a3` | 引数レジスタ | 関数の引数渡し |
| `$v0-$v1` | 戻り値レジスタ | 関数の戻り値 |

---

## 命令分類と学習のコツ

### 1. 算術演算（覚えやすい基本形）

#### 基本パターン: `命令 $dest, $src1, $src2`
```assembly
add  $s1, $s2, $s3    # $s1 = $s2 + $s3
sub  $s1, $s2, $s3    # $s1 = $s2 - $s3
```

#### 即値演算: `命令i $dest, $src, 定数`
```assembly
addi $s1, $s2, 20     # $s1 = $s2 + 20
```

**覚え方**: "i" = immediate（即値）

---

### 2. データ転送（メモリとレジスタ間）

#### ロード: メモリ → レジスタ
```assembly
lw   $s1, 20($s2)     # $s1 = メモリ[$s2+20] (4バイト)
lh   $s1, 20($s2)     # $s1 = メモリ[$s2+20] (2バイト、符号拡張)
lhu  $s1, 20($s2)     # $s1 = メモリ[$s2+20] (2バイト、ゼロ拡張)
lb   $s1, 20($s2)     # $s1 = メモリ[$s2+20] (1バイト、符号拡張)
lbu  $s1, 20($s2)     # $s1 = メモリ[$s2+20] (1バイト、ゼロ拡張)
```

#### ストア: レジスタ → メモリ
```assembly
sw   $s1, 20($s2)     # メモリ[$s2+20] = $s1 (4バイト)
sh   $s1, 20($s2)     # メモリ[$s2+20] = $s1 (2バイト)
sb   $s1, 20($s2)     # メモリ[$s2+20] = $s1 (1バイト)
```

**覚え方**: 
- l = load（読み込み）, s = store（保存）
- w = word（4バイト）, h = half（2バイト）, b = byte（1バイト）
- u = unsigned（符号なし拡張）

---

### 3. 論理演算

#### ビット演算
```assembly
and  $s1, $s2, $s3    # $s1 = $s2 & $s3 (ビット単位AND)
or   $s1, $s2, $s3    # $s1 = $s2 | $s3 (ビット単位OR)
nor  $s1, $s2, $s3    # $s1 = ~($s2 | $s3) (ビット単位NOR)
```

#### 即値との論理演算
```assembly
andi $s1, $s2, 20     # $s1 = $s2 & 20
ori  $s1, $s2, 20     # $s1 = $s2 | 20
```

#### シフト演算
```assembly
sll  $s1, $s2, 10     # $s1 = $s2 << 10 (左シフト)
srl  $s1, $s2, 10     # $s1 = $s2 >> 10 (右シフト、ゼロ埋め)
```

**覚え方**: 
- sll = shift left logical, srl = shift right logical

---

### 4. 条件分岐

#### 等しい/等しくない
```assembly
beq  $s1, $s2, 25     # if($s1 == $s2) goto PC+4+100
bne  $s1, $s2, 25     # if($s1 != $s2) goto PC+4+100
```

#### 大小比較 + セット
```assembly
slt  $s1, $s2, $s3    # if($s2 < $s3) $s1=1; else $s1=0
sltu $s1, $s2, $s3    # 符号なし数での比較
slti $s1, $s2, 20     # if($s2 < 20) $s1=1; else $s1=0
```

**覚え方**: 
- beq = branch equal, bne = branch not equal
- slt = set less than
- u = unsigned（符号なし）

---

### 5. 無条件ジャンプ

```assembly
j    2500             # goto 10000番地
jr   $ra              # goto $ra (関数復帰)
jal  2500             # $ra=PC+4; goto 10000 (関数呼び出し)
```

**覚え方**: 
- j = jump, jr = jump register, jal = jump and link

---

## 実践的な学習法

### 1. パターン認識で覚える

#### R形式（レジスタ3個）
```
[演算子] $dest, $src1, $src2
例: add, sub, and, or, slt
```

#### I形式（即値あり）
```
[演算子]i $dest, $src, 定数
例: addi, andi, ori, slti

[ロード/ストア] $reg, offset($base)
例: lw, sw, lh, sh, lb, sb
```

#### J形式（ジャンプ）
```
[ジャンプ] アドレス
例: j, jal
```

### 2. 頻出パターンを覚える

#### 配列アクセス
```assembly
# array[i] を $t0 に読み込み（intの場合）
sll $t1, $s1, 2      # $t1 = i * 4 (iを4倍)
add $t2, $s0, $t1    # $t2 = &array[0] + i*4
lw  $t0, 0($t2)      # $t0 = array[i]
```

#### if文の実装
```assembly
# if (a == b) { ... }
bne $s0, $s1, else   # if (a != b) goto else
# then部分の処理
# ...
else:
# else部分の処理
```

#### for文の実装
```assembly
# for (i = 0; i < n; i++)
    add $t0, $zero, $zero  # i = 0
loop:
    beq $t0, $s0, exit     # if (i == n) goto exit
    # ループ本体
    addi $t0, $t0, 1       # i++
    j loop                 # goto loop
exit:
```

### 3. 覚えるべき優先順位

#### 最重要（必須）
- `add, addi, sub` (算術)
- `lw, sw` (メモリアクセス)
- `beq, bne` (分岐)
- `j, jr` (ジャンプ)

#### 重要
- `and, or, andi, ori` (論理)
- `slt, slti` (比較)
- `sll, srl` (シフト)

#### 応用
- `lh, lhu, lb, lbu, sh, sb` (様々なサイズのメモリアクセス)
- `sltu, sltiu` (符号なし比較)
- `jal` (関数呼び出し)

---

## クイズで確認

### 問題1: 基本演算
```assembly
# C言語: c = a + b - 5;
# $s0=a, $s1=b, $s2=cとして書け
```

<details>
<summary>解答</summary>

```assembly
add $t0, $s0, $s1    # $t0 = a + b
addi $s2, $t0, -5    # $s2 = (a + b) - 5
```

または

```assembly
add $s2, $s0, $s1    # $s2 = a + b
addi $s2, $s2, -5    # $s2 = $s2 - 5
```

</details>

### 問題2: 配列アクセス
```assembly
# C言語: x = array[3];
# $s0=&array[0], $s1=xとして書け
```

<details>
<summary>解答</summary>

```assembly
lw $s1, 12($s0)      # $s1 = array[3] (3*4=12バイトオフセット)
```

</details>

### 問題3: 条件分岐
```assembly
# C言語: if (a < b) x = 1; else x = 0;
# $s0=a, $s1=b, $s2=xとして書け
```

<details>
<summary>解答</summary>

```assembly
slt $s2, $s0, $s1    # if(a < b) $s2=1; else $s2=0
```

</details>

---

## まとめ

MIPSの命令セットは**パターン化**されているので、規則性を理解すれば効率的に覚えられます：

1. **形式別に分類**して覚える（R形式、I形式、J形式）
2. **頻出パターン**を実際のコードで練習する
3. **接尾辞の意味**を理解する（i=immediate, u=unsigned等）
4. **C言語との対応**で実践的に学習する

TypeScriptでの開発経験があれば、メモリ管理や低レベル操作の概念として理解しやすいはずです！