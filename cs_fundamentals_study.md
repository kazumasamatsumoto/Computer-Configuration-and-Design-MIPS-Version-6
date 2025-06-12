# 🖥️ コンピュータサイエンス基礎：文字エンコーディングとアセンブリ言語

## 📊 ASCII (American Standard Code for Information Interchange)

### 🎯 原理原則

**ASCII**は 7 ビット（128 文字）の文字エンコーディング標準です。コンピュータが文字を数値として処理するための基礎となります。

#### 基本仕様

- **数値範囲:** 0-127 (7 ビット)
- **構成:** 制御文字(0-31) + 印刷可能文字(32-126) + DEL(127)
- **重要な範囲:**
  - 'A'-'Z': 65-90
  - 'a'-'z': 97-122
  - '0'-'9': 48-57

### 主要 ASCII 文字表

| 文字 | コード | 文字 | コード | 文字 | コード | 文字 | コード |
| ---- | ------ | ---- | ------ | ---- | ------ | ---- | ------ |
| SP   | 32     | 0    | 48     | @    | 64     | P    | 80     |
| !    | 33     | 1    | 49     | A    | 65     | Q    | 81     |
| "    | 34     | 2    | 50     | B    | 66     | R    | 82     |
| #    | 35     | 3    | 51     | C    | 67     | S    | 83     |
| $    | 36     | 4    | 52     | D    | 68     | T    | 84     |
| %    | 37     | 5    | 53     | E    | 69     | U    | 85     |
| &    | 38     | 6    | 54     | F    | 70     | V    | 86     |
| '    | 39     | 7    | 55     | G    | 71     | W    | 87     |
| (    | 40     | 8    | 56     | H    | 72     | X    | 88     |
| )    | 41     | 9    | 57     | I    | 73     | Y    | 89     |
| \*   | 42     | :    | 58     | J    | 74     | Z    | 90     |
| +    | 43     | ;    | 59     | K    | 75     | [    | 91     |
| ,    | 44     | <    | 60     | L    | 76     | \    | 92     |
| -    | 45     | =    | 61     | M    | 77     | ]    | 93     |
| .    | 46     | >    | 62     | N    | 78     | ^    | 94     |
| /    | 47     | ?    | 63     | O    | 79     | \_   | 95     |
| a    | 97     | h    | 104    | o    | 111    | v    | 118    |
| b    | 98     | i    | 105    | p    | 112    | w    | 119    |
| c    | 99     | j    | 106    | q    | 113    | x    | 120    |
| d    | 100    | k    | 107    | r    | 114    | y    | 121    |
| e    | 101    | l    | 108    | s    | 115    | z    | 122    |
| f    | 102    | m    | 109    | t    | 116    | {    | 123    |
| g    | 103    | n    | 110    | u    | 117    | }    | 125    |

> 💡 **重要なポイント:** 大文字と小文字の差は 32 です（'A'=65, 'a'=97, 97-65=32）

---

## ⚙️ MIPS アセンブリ言語

### 🎯 原理原則

**MIPS**は RISC (Reduced Instruction Set Computer) アーキテクチャの代表例です。

#### 基本仕様

- **レジスタ:** 32 個の汎用レジスタ ($0-$31)
- **命令形式:** 固定長 32 ビット
- **メモリアクセス:** Load/Store アーキテクチャ

### 文字列コピー関数の例

以下は C 言語の `void strcpy(char x[], char y[])` を MIPS アセンブリで実装した例です：

```assembly
# C言語: void strcpy(char x[], char y[])
strcpy:
    addi $sp, $sp, -4     # スタックに1語分のスペースを取る
    sw   $s0, 0($sp)      # $s0 を退避

    add  $s0, $zero, $zero # i = 0

L1:
    add  $t1, $s0, $a1     # y[i]のアドレスを$t1に代入
    lbu  $t2, 0($t1)      # $t2 = y[i]
    add  $t3, $s0, $a0     # x[i]のアドレスを$t3に代入
    sb   $t2, 0($t3)      # x[i] = y[i]
    beq  $t2, $zero, L2   # y[i]がゼロなら L2に分岐
    addi $s0, $s0, 1      # i = i + 1
    j    L1               # L1へジャンプ

L2:
    lw   $s0, 0($sp)      # y[i] == 0, 文字列の終了
    addi $sp, $sp, 4      # スタックを1語分ポップする
    jr   $ra              # 呼出元に戻る
```

### 重要な命令の説明

| 命令   | 意味               | 説明                                   |
| ------ | ------------------ | -------------------------------------- |
| `lbu`  | Load Byte Unsigned | バイトを符号なしでロード               |
| `sb`   | Store Byte         | バイトをストア                         |
| `beq`  | Branch if Equal    | 等しい場合に分岐                       |
| `addi` | Add Immediate      | 即値を加算                             |
| `jr`   | Jump Register      | レジスタで指定されたアドレスにジャンプ |

> 💡 **重要なポイント:**
>
> - 文字列は null 終端（'\0'）で終了
> - バイト単位でのメモリアクセスが必要
> - スタックでレジスタの値を保存・復元

---

## 🌐 Unicode と文字体系

### 🎯 原理原則

**Unicode**は世界中の文字体系を統一的にエンコードする標準です。

#### 基本仕様

- **コードポイント:** U+0000 から U+10FFFF まで
- **エンコーディング:** UTF-8, UTF-16, UTF-32
- **ASCII 互換:** U+0000-U+007F は ASCII と同じ

### 世界の文字体系と Unicode 範囲

#### 🔤 ラテン文字系

- **基本ラテン:** U+0000-U+007F
- **ラテン拡張 A:** U+0100-U+017F
- **ラテン拡張 B:** U+0180-U+024F

#### 🈲 CJK 文字系

- **ひらがな:** U+3040-U+309F
- **カタカナ:** U+30A0-U+30FF
- **漢字:** U+4E00-U+9FFF

#### 🌍 その他の文字系

- **アラビア文字:** U+0600-U+06FF
- **キリル文字:** U+0400-U+04FF
- **デーヴァナーガリー:** U+0900-U+097F

### UTF-8 エンコーディングの仕組み

| コードポイント範囲 | UTF-8 エンコーディング              | バイト数 |
| ------------------ | ----------------------------------- | -------- |
| U+0000 - U+007F    | 0xxxxxxx                            | 1 バイト |
| U+0080 - U+07FF    | 110xxxxx 10xxxxxx                   | 2 バイト |
| U+0800 - U+FFFF    | 1110xxxx 10xxxxxx 10xxxxxx          | 3 バイト |
| U+10000 - U+10FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx | 4 バイト |

> 💡 **重要なポイント:** UTF-8 は ASCII 後方互換性を保ちながら、全世界の文字を表現できます。

---

## 🧠 理解度クイズ

### Q1. ASCII 文字'A'のコードは何ですか？

**選択肢:**
a) 64  
b) 65  
c) 66  
d) 97

**答え: b) 65**

**解説:** ASCII 文字'A'は 65 です。大文字 A-Z は 65-90 の範囲にあります。

### Q2. MIPS 命令「lbu」の意味は？

**選択肢:**
a) Load Byte  
b) Load Binary Unit  
c) Load Byte Unsigned  
d) Load Base Unit

**答え: c) Load Byte Unsigned**

**解説:** 「lbu」は Load Byte Unsigned の略で、メモリから 1 バイトを符号なしでレジスタにロードします。

### Q3. 文字列"Cat"を ASCII 数値列で表すと？

**選択肢:**
a) 65, 97, 116  
b) 67, 97, 116  
c) 67, 65, 116  
d) 99, 97, 116

**答え: b) 67, 97, 116**

**解説:** 'C'=67, 'a'=97, 't'=116 です。大文字 C=67、小文字 a=97、小文字 t=116 となります。

### Q4. UTF-8 で 1 バイトで表現できる文字の範囲は？

**選択肢:**
a) U+0000 - U+007F  
b) U+0000 - U+00FF  
c) U+0000 - U+07FF  
d) U+0000 - U+FFFF

**答え: a) U+0000 - U+007F**

**解説:** UTF-8 で 1 バイト（7 ビット）で表現できるのは U+0000-U+007F、つまり ASCII 文字と同じ範囲です。

### Q5. MIPS の文字列コピーで、ループ終了条件は何ですか？

**選択肢:**
a) 文字列の長さが指定値に達したとき  
b) メモリが不足したとき  
c) レジスタがオーバーフローしたとき  
d) null 文字（'\0'）に遭遇したとき

**答え: d) null 文字（'\0'）に遭遇したとき**

**解説:** C 言語の文字列は null 終端なので、'\0'（ASCII 値 0）に遭遇するとループを終了します。

---

## 💡 TypeScript/Angular 開発への応用

### 文字エンコーディング知識の活用

#### 1. **国際化対応 (i18n)**

```typescript
// Unicode正規化を使用した検索機能
function normalizeString(str: string): string {
  return str.normalize("NFC"); // 合成文字を正規化
}

// 日本語文字の判定
function isJapanese(char: string): boolean {
  const code = char.codePointAt(0);
  return (
    (code >= 0x3040 && code <= 0x309f) || // ひらがな
    (code >= 0x30a0 && code <= 0x30ff) || // カタカナ
    (code >= 0x4e00 && code <= 0x9faf)
  ); // 漢字
}
```

#### 2. **文字列処理の最適化**

```typescript
// ASCII文字の高速判定
function isAscii(str: string): boolean {
  for (let i = 0; i < str.length; i++) {
    if (str.charCodeAt(i) > 127) return false;
  }
  return true;
}

// 大文字小文字変換の原理理解
function toUpperCaseAscii(char: string): string {
  const code = char.charCodeAt(0);
  if (code >= 97 && code <= 122) {
    // 'a'-'z'
    return String.fromCharCode(code - 32);
  }
  return char;
}
```

#### 3. **バイナリデータ処理**

```typescript
// UTF-8バイト列の処理
function utf8ByteLength(str: string): number {
  return new TextEncoder().encode(str).length;
}

// Base64エンコーディング理解
function base64Encode(str: string): string {
  return btoa(unescape(encodeURIComponent(str)));
}
```

### 低レベル知識がもたらす利点

1. **パフォーマンス最適化:** 文字エンコーディングの理解により効率的な文字列処理
2. **国際化対応:** Unicode 知識によるグローバルアプリケーション開発
3. **データ処理:** バイナリデータやファイル形式の正確な処理
4. **デバッグ能力:** 文字化けや文字列関連バグの迅速な特定

これらの基礎知識は、Angular/TypeScript での高度な文字列処理、国際化対応、パフォーマンス最適化において重要な土台となります。
