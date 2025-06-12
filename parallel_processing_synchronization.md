# 並列処理と同期制御 完全ガイド

## 概要

並列処理において、複数のプロセッサやスレッドが同じデータにアクセスする際に発生する**データ競合（Data Race）**を防ぐための同期機構について学習します。特にMIPSアーキテクチャにおける原子的操作とロック機構を中心に解説します。

---

## 1. 並列処理における課題

### データ競合（Data Race）とは

**定義**: 複数のスレッドが同じメモリ位置に対して、少なくとも1つが書き込み操作を行う際に、適切な同期なしにアクセスすることで発生する問題

**具体例**:
```c
// 共有変数
int counter = 0;

// スレッド1
counter = counter + 1;

// スレッド2  
counter = counter + 1;
```

**期待値**: counter = 2
**実際の結果**: counter = 1 または 2 （不定）

### なぜデータ競合が起こるのか？

機械語レベルでの動作を見てみましょう：

```assembly
# counter++ の機械語展開
lw   $t0, counter($gp)    # メモリから値をロード
addi $t0, $t0, 1          # 値を1増加
sw   $t0, counter($gp)    # メモリに書き戻し
```

この3命令の間に他のプロセッサが割り込むと、データ競合が発生します。

---

## 2. 同期プリミティブ

### 2.1 ロック（Lock）とアンロック（Unlock）

**基本概念**:
- **ロック**: 共有リソースへの排他的アクセス権を取得
- **アンロック**: アクセス権を解放

**理想的な動作**:
```
プロセッサA: lock → critical section → unlock
プロセッサB:        待機            → lock → critical section → unlock
```

### 2.2 相互排除（Mutual Exclusion）

**定義**: 同時に実行できるプロセスは最大1つまでという制約

**実装要件**:
1. **相互排除**: 複数のプロセスが同時にクリティカルセクションに入らない
2. **進行保証**: 待機中のプロセスは最終的にクリティカルセクションに入れる
3. **有界待機**: 無限に待機することがない

---

## 3. MIPSにおける原子的操作

### 3.1 Load Linked（ll）命令

**機能**: メモリから値を読み込み、そのアドレスに「リンク」を設定

**構文**: `ll $rt, offset($rs)`

**動作**:
1. メモリアドレス `$rs + offset` から値を読み込み
2. そのアドレスをプロセッサの「リンクレジスタ」に記録
3. 値を `$rt` に格納

### 3.2 Store Conditional（sc）命令

**機能**: リンクが有効な場合のみメモリに書き込み

**構文**: `sc $rt, offset($rs)`

**動作**:
1. リンクが有効（他のプロセッサが該当アドレスを変更していない）
   - メモリに `$rt` の値を書き込み
   - `$rt` に 1 を設定（成功）
2. リンクが無効
   - 書き込みを実行しない
   - `$rt` に 0 を設定（失敗）

### 3.3 原子的交換（Atomic Exchange）の実装

```assembly
# atomic_exchange(&lock, 1) の実装
atomic_exchange:
    li    $t0, 1              # 新しい値（ロック状態）を準備

exchange_loop:
    ll    $v0, 0($a0)         # 現在の値をロード（リンク設定）
    sc    $t0, 0($a0)         # 条件付きストア
    beq   $t0, $zero, exchange_loop  # 失敗時は再試行
    jr    $ra                 # 成功時は復帰（$v0に古い値）
```

---

## 4. スピンロックの実装

### 4.1 基本的なスピンロック

```assembly
# スピンロック取得
acquire_lock:
    li    $t0, 1              # ロック値（1 = ロック中）

spin_wait:
    ll    $t1, 0($a0)         # ロック変数をロード
    bne   $t1, $zero, spin_wait # ロック中なら待機
    sc    $t0, 0($a0)         # ロック取得を試行
    beq   $t0, $zero, spin_wait # 失敗なら再試行
    jr    $ra                 # 成功時は復帰

# スピンロック解放  
release_lock:
    sw    $zero, 0($a0)       # ロック変数を0に設定
    jr    $ra
```

### 4.2 使用例：カウンタのインクリメント

```assembly
.data
counter: .word 0
lock:    .word 0

.text
increment_counter:
    # ロック取得
    la    $a0, lock
    jal   acquire_lock
    
    # クリティカルセクション
    la    $t0, counter
    lw    $t1, 0($t0)
    addi  $t1, $t1, 1
    sw    $t1, 0($t0)
    
    # ロック解放
    la    $a0, lock
    jal   release_lock
    
    jr    $ra
```

---

## 5. より高度な同期プリミティブ

### 5.1 Compare-And-Swap（CAS）

```assembly
# compare_and_swap(address, expected, new_value)
# 戻り値: 成功時1、失敗時0
compare_and_swap:
    ll    $t0, 0($a0)         # 現在値をロード
    bne   $t0, $a1, cas_fail  # 期待値と異なれば失敗
    sc    $a2, 0($a0)         # 新しい値をストア
    move  $v0, $a2            # 結果を返す
    jr    $ra

cas_fail:
    li    $v0, 0              # 失敗を示す
    jr    $ra
```

### 5.2 Fetch-And-Add

```assembly
# fetch_and_add(address, increment)
# 戻り値: 古い値
fetch_and_add:
fetch_loop:
    ll    $v0, 0($a0)         # 現在値をロード
    add   $t0, $v0, $a1       # インクリメント値を加算
    sc    $t0, 0($a0)         # 条件付きストア
    beq   $t0, $zero, fetch_loop # 失敗時は再試行
    jr    $ra                 # 古い値を返す
```

---

## 6. パフォーマンスとスケーラビリティ

### 6.1 スピンロックの問題点

1. **CPUサイクルの浪費**: 待機中もCPUを消費
2. **キャッシュライン競合**: 複数プロセッサが同じキャッシュラインにアクセス
3. **優先度逆転**: 低優先度プロセスがロックを保持

### 6.2 改善技法

**指数バックオフ**:
```assembly
spin_with_backoff:
    li    $t2, 1              # 初期待機時間

retry_with_delay:
    ll    $t1, 0($a0)
    bne   $t1, $zero, backoff
    sc    $t0, 0($a0)
    bne   $t0, $zero, success
    
backoff:
    move  $t3, $t2            # 待機ループ
delay_loop:
    subi  $t3, $t3, 1
    bne   $t3, $zero, delay_loop
    
    sll   $t2, $t2, 1         # 待機時間を倍増
    j     retry_with_delay

success:
    jr    $ra
```

---

## 7. TypeScript/Node.jsとの関連

### 7.1 JavaScript の同期プリミティブ

```typescript
// SharedArrayBuffer を使用した原子的操作
const sharedBuffer = new SharedArrayBuffer(1024);
const sharedArray = new Int32Array(sharedBuffer);

// 原子的比較交換
const oldValue = Atomics.compareExchange(sharedArray, 0, expectedValue, newValue);

// 原子的加算
const oldValue = Atomics.add(sharedArray, 0, 1);

// ロック風の実装
class SpinLock {
    private lockIndex = 0;
    
    constructor(private sharedArray: Int32Array) {}
    
    acquire(): void {
        while (true) {
            const acquired = Atomics.compareExchange(
                this.sharedArray, 
                this.lockIndex, 
                0, // expected: unlocked
                1  // new: locked
            );
            if (acquired === 0) break; // 成功
            // スピン待機（実際にはyield推奨）
        }
    }
    
    release(): void {
        Atomics.store(this.sharedArray, this.lockIndex, 0);
    }
}
```

### 7.2 Node.js Worker Threads での応用

```typescript
// main.ts
import { Worker, isMainThread, parentPort, workerData } from 'worker_threads';

if (isMainThread) {
    const sharedBuffer = new SharedArrayBuffer(1024);
    const sharedArray = new Int32Array(sharedBuffer);
    
    // 複数ワーカーを起動
    for (let i = 0; i < 4; i++) {
        new Worker(__filename, {
            workerData: { sharedBuffer }
        });
    }
} else {
    // Worker内での処理
    const sharedArray = new Int32Array(workerData.sharedBuffer);
    const lock = new SpinLock(sharedArray);
    
    // 同期が必要な処理
    lock.acquire();
    try {
        // クリティカルセクション
        const counter = Atomics.load(sharedArray, 1);
        Atomics.store(sharedArray, 1, counter + 1);
    } finally {
        lock.release();
    }
}
```

---

## 8. 衝突（コンフリクト）発生時の詳細動作

### 8.1 ll/sc命令における衝突パターン

ll/sc命令ペアでリンクが無効化される具体的なケースを見てみましょう。

#### パターン1: 他プロセッサによる書き込み衝突

```assembly
# プロセッサA（時刻t1）
ll    $t0, 0($s0)        # アドレス0x1000から値5をロード、リンク設定
                         # リンク: 0x1000 → 有効

# プロセッサB（時刻t2）
sw    $t1, 0($s0)        # 同じアドレス0x1000に書き込み
                         # → プロセッサAのリンクが無効化される

# プロセッサA（時刻t3）
addi  $t0, $t0, 1        # 5 + 1 = 6
sc    $t0, 0($s0)        # 条件付きストア実行
                         # リンク無効のため失敗
                         # $t0 = 0（失敗フラグ）
                         # メモリは変更されない
```

#### パターン2: 同一アドレスへの複数ll命令

```assembly
# プロセッサA
ll    $t0, 0($s0)        # アドレス0x1000をリンク

# プロセッサB  
ll    $t1, 0($s0)        # 同じアドレスをリンク
                         # → プロセッサAのリンクが無効化される場合がある
                         # （実装依存）

# プロセッサA
sc    $t0, 0($s0)        # 失敗する可能性
```

### 8.2 時系列での詳細な衝突例

**シナリオ**: 2つのプロセッサが同時にカウンタをインクリメント

```assembly
# 初期状態: counter = 10

# === 時刻 T1 ===
# プロセッサA
ll    $t0, counter       # $t0 = 10, リンクA設定

# プロセッサB  
ll    $t1, counter       # $t1 = 10, リンクB設定（リンクAは無効化）

# === 時刻 T2 ===
# プロセッサA
addi  $t0, $t0, 1        # $t0 = 11

# プロセッサB
addi  $t1, $t1, 1        # $t1 = 11

# === 時刻 T3 ===
# プロセッサB（先に実行）
sc    $t1, counter       # 成功: counter = 11, $t1 = 1

# === 時刻 T4 ===  
# プロセッサA
sc    $t0, counter       # 失敗: counter は変更されず, $t0 = 0
                         # リンクAは既に無効
```

### 8.3 具体的な衝突検出メカニズム

```assembly
# 衝突を考慮した完全な原子的インクリメント
atomic_increment:
    la    $a0, counter

retry_loop:
    ll    $t0, 0($a0)        # 現在値をロード、リンク設定
    
    # === この間に他プロセッサが割り込む可能性 ===
    # 他プロセッサ: sw $tx, 0($a0) → リンク無効化
    
    addi  $t0, $t0, 1        # 値をインクリメント
    sc    $t0, 0($a0)        # 条件付きストア
    
    beq   $t0, $zero, retry_loop  # 失敗時（$t0=0）は再試行
    
    # 成功時のみここに到達
    jr    $ra

# 衝突状況のデバッグ版
atomic_increment_debug:
    la    $a0, counter
    li    $t9, 0             # 試行回数カウンタ

retry_with_count:
    addi  $t9, $t9, 1        # 試行回数増加
    
    ll    $t0, 0($a0)        # ロード＆リンク
    addi  $t0, $t0, 1        # インクリメント
    sc    $t0, 0($a0)        # 条件付きストア
    
    bne   $t0, $zero, success # 成功時は終了
    
    # 失敗時の処理
    move  $a1, $t9
    jal   print_retry_count   # 「試行 X 回目で衝突」を出力
    j     retry_with_count

success:
    move  $v0, $t9           # 総試行回数を返す
    jr    $ra
```

### 8.4 高競合環境での衝突パターン

```assembly
# 4つのプロセッサが同時実行する場合
.data
shared_counter: .word 100
collision_stats: .word 0, 0, 0, 0  # 各プロセッサの衝突回数

.text
# プロセッサID = $a1（0-3）
contended_increment:
    move  $t8, $a1           # プロセッサID保存
    la    $t7, collision_stats
    sll   $t6, $t8, 2        # ID * 4（ワードサイズ）
    add   $t7, $t7, $t6      # 自プロセッサの統計アドレス

attempt_increment:
    ll    $t0, shared_counter
    
    # 人工的な遅延（衝突確率を上げる）
    li    $t5, 100
delay_loop:
    subi  $t5, $t5, 1
    bne   $t5, $zero, delay_loop
    
    addi  $t0, $t0, 1
    sc    $t0, shared_counter
    
    bne   $t0, $zero, increment_success
    
    # 衝突発生時の統計更新
    lw    $t4, 0($t7)        # 現在の衝突回数
    addi  $t4, $t4, 1        # インクリメント
    sw    $t4, 0($t7)        # 書き戻し
    
    j     attempt_increment   # 再試行

increment_success:
    jr    $ra
```

### 8.5 リンク無効化の詳細条件

MIPSアーキテクチャでは、以下の場合にリンクが無効化されます：

1. **他プロセッサによる書き込み**
   ```assembly
   # プロセッサA
   ll $t0, 0($s0)     # リンク設定
   
   # プロセッサB
   sw $t1, 0($s0)     # → Aのリンク無効化
   ```

2. **例外やコンテキストスイッチ**
   ```assembly
   ll $t0, 0($s0)     # リンク設定
   syscall            # システムコール → リンク無効化
   sc $t0, 0($s0)     # 必ず失敗
   ```

3. **キャッシュラインの追い出し**
   ```assembly
   ll $t0, 0($s0)     # リンク設定
   # 大量のメモリアクセスでキャッシュライン追い出し
   lw $t1, 0($s1)     # 異なるアドレスだがキャッシュ衝突
   # ...
   sc $t0, 0($s0)     # 失敗の可能性
   ```

### 8.6 TypeScriptでの衝突シミュレーション

```typescript
class LLSCSimulator {
    private memory: Map<number, number> = new Map();
    private links: Map<number, number> = new Map(); // processor -> address
    
    // Load Linked simulation
    ll(processorId: number, address: number): number {
        const value = this.memory.get(address) || 0;
        this.links.set(processorId, address);
        console.log(`P${processorId}: ll addr=${address}, value=${value}, link set`);
        return value;
    }
    
    // Store Conditional simulation  
    sc(processorId: number, address: number, value: number): boolean {
        const linkedAddr = this.links.get(processorId);
        
        if (linkedAddr !== address) {
            console.log(`P${processorId}: sc FAILED - no link for addr=${address}`);
            return false;
        }
        
        // Check if other processors have linked to same address
        const otherLinks = Array.from(this.links.entries())
            .filter(([pid, addr]) => pid !== processorId && addr === address);
            
        if (otherLinks.length > 0) {
            console.log(`P${processorId}: sc FAILED - conflict with processors: ${otherLinks.map(([pid]) => pid)}`);
            this.links.delete(processorId);
            return false;
        }
        
        this.memory.set(address, value);
        this.links.delete(processorId);
        console.log(`P${processorId}: sc SUCCESS - addr=${address}, value=${value}`);
        return true;
    }
    
    // Simulate write by another processor
    write(processorId: number, address: number, value: number): void {
        // Invalidate all links to this address
        const affectedProcessors: number[] = [];
        for (const [pid, linkedAddr] of this.links) {
            if (linkedAddr === address && pid !== processorId) {
                this.links.delete(pid);
                affectedProcessors.push(pid);
            }
        }
        
        this.memory.set(address, value);
        console.log(`P${processorId}: write addr=${address}, value=${value}, invalidated links: ${affectedProcessors}`);
    }
}

// 衝突シナリオのシミュレーション
const sim = new LLSCSimulator();

// シナリオ1: 基本的な衝突
console.log("=== Scenario 1: Basic Conflict ===");
sim.ll(0, 0x1000);        // P0: ll
sim.ll(1, 0x1000);        // P1: ll (P0のリンクを無効化)
sim.sc(0, 0x1000, 42);    // P0: sc (失敗)
sim.sc(1, 0x1000, 43);    // P1: sc (成功)

// シナリオ2: 書き込み割り込み
console.log("\n=== Scenario 2: Write Interruption ===");
sim.ll(0, 0x2000);        // P0: ll
sim.write(1, 0x2000, 99); // P1: 通常書き込み (P0のリンクを無効化)
sim.sc(0, 0x2000, 50);    // P0: sc (失敗)
```

### 8.7 実際の衝突対策コード

```assembly
# 指数バックオフによる衝突軽減
atomic_increment_backoff:
    li    $t9, 1             # 初期バックオフ時間
    li    $t8, 16            # 最大バックオフ時間

retry_with_backoff:
    ll    $t0, 0($a0)
    addi  $t0, $t0, 1
    sc    $t0, 0($a0)
    bne   $t0, $zero, success
    
    # バックオフ実行
    move  $t7, $t9
backoff_loop:
    nop                      # 待機
    subi  $t7, $t7, 1
    bne   $t7, $zero, backoff_loop
    
    # バックオフ時間を倍増（上限あり）
    sll   $t9, $t9, 1        # * 2
    ble   $t9, $t8, retry_with_backoff
    move  $t9, $t8           # 上限でクリップ
    j     retry_with_backoff

success:
    jr    $ra
```

---

## 9. 理解度チェッククイズ

### 【基礎レベル】

**問題1**: Load Linked（ll）命令の主な目的は何ですか？

<details>
<summary>解答</summary>

**メモリアドレスへのリンクを設定し、その後の変更を検出可能にする**

解説: ll命令は値の読み込みと同時に、そのアドレスを監視対象として登録します。これにより、他のプロセッサによる変更を検出できます。
</details>

**問題2**: Store Conditional（sc）命令が失敗する条件は？

<details>
<summary>解答</summary>

**リンクが無効になっている場合（他のプロセッサが該当アドレスを変更した場合）**

解説: ll命令で設定したリンクが、他のプロセッサによる書き込みや特定の例外によって無効化されると、sc命令は失敗します。
</details>

### 【応用レベル】

**問題3**: 次のコードの問題点を指摘し、修正案を示してください。

```assembly
# 不完全なスピンロック実装
acquire_bad_lock:
    lw    $t0, 0($a0)        # ロック変数をロード
    bne   $t0, $zero, acquire_bad_lock  # ロック中なら待機
    li    $t1, 1
    sw    $t1, 0($a0)        # ロックを設定
    jr    $ra
```

<details>
<summary>解答</summary>

**問題点**: 
- ロード〜ストア間に他のプロセッサが割り込む可能性
- 原子的操作になっていない

**修正案**:
```assembly
acquire_lock:
    li    $t0, 1
spin_wait:
    ll    $t1, 0($a0)
    bne   $t1, $zero, spin_wait
    sc    $t0, 0($a0)
    beq   $t0, $zero, spin_wait
    jr    $ra
```

解説: ll/sc命令ペアを使用することで、原子的なテスト&セット操作を実現します。
</details>

**問題4**: デッドロック回避のための基本戦略を2つ挙げてください。

<details>
<summary>解答</summary>

1. **ロック順序の統一**: 全てのスレッドが同じ順序でロックを取得
2. **タイムアウト機構**: 一定時間でロック取得を諦める

その他の戦略:
- 銀行家のアルゴリズム
- リソースの事前割り当て
- 循環待機の回避
</details>

### 【発展レベル】

**問題5**: 以下のMIPSコードが実装している同期プリミティブは何か、また、その動作原理を説明してください。

```assembly
mystery_sync:
    move  $t2, $a1           # 第2引数を保存
retry:
    ll    $t0, 0($a0)        # アドレス$a0から値をロード
    add   $t1, $t0, $t2      # 読み込んだ値に$t2を加算
    sc    $t1, 0($a0)        # 結果を条件付きストア
    beq   $t1, $zero, retry  # 失敗時は再試行
    move  $v0, $t0           # 元の値を返す
    jr    $ra
```

<details>
<summary>解答</summary>

**同期プリミティブ**: Fetch-And-Add（フェッチ＆アド）

**動作原理**:
1. メモリ位置から現在値を原子的に読み込み（ll）
2. 指定された値を加算
3. 他のプロセッサによる変更がなければ結果を書き込み（sc）
4. 変更があった場合は最初からやり直し
5. 操作前の値を返す

**用途**: 原子的カウンタ、統計情報の更新など

**TypeScript対応**:
```typescript
const oldValue = Atomics.add(sharedArray, index, increment);
```
</details>

---

## 9. 実践演習

### 演習1: プロデューサー・コンシューマー問題

リングバッファを使用した生産者・消費者パターンをMIPSアセンブリで実装してください。

**要件**:
- 固定サイズのリングバッファ
- 複数の生産者と消費者
- オーバーフロー/アンダーフロー防止

### 演習2: Read-Write Lock

読み書きロックをll/sc命令を使って実装してください。

**要件**:
- 複数の読み手は同時実行可能
- 書き手は排他的実行
- 書き手の優先度考慮

---

## 10. まとめ

並列処理における同期制御は、現代のマルチコア・システムにおいて不可欠な技術です：

### 重要なポイント

1. **原子性の重要性**: 単純な操作でも機械語レベルでは複数命令に分解される
2. **ハードウェア支援**: ll/sc命令による効率的な同期プリミティブ
3. **パフォーマンス考慮**: スピンロックのオーバーヘッドとスケーラビリティ
4. **高レベル言語との対応**: TypeScript/JavaScriptの原子的操作との関連

### 次のステップ

- キャッシュコヒーレンシプロトコル
- メモリバリア・フェンス
- Lock-freeデータ構造
- マルチプロセッサスケジューリング

これらの概念は、フロントエンド開発でのWorker Threadsや、Node.jsでの並行処理においても重要な基盤知識となります。