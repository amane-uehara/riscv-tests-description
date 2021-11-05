# riscv-tests の内部構造について

<https://github.com/riscv-software-src/riscv-tests/>

の内部構造を追跡していたのでメモを残しておく。

以下では`rv32ui`の加算命令のテストがどのように動くか見ていく。
仮想メモリは使用せず、コア数は1個で、割込が入らない状況を考える。

# テストコードの呼び出し部分

表面部分から順番に掘り下げて見ていく。

### `isa/rv32ui/add.S`

<https://github.com/riscv-software-src/riscv-tests/blob/e65ecdf941a5484af27f9be223fb655ebcb0398b/isa/rv32ui/add.S>

```
#include "riscv_test.h"
#undef RVTEST_RV64U
#define RVTEST_RV64U RVTEST_RV32U

#include "../rv64ui/add.S"
```

`include`可能な`riscv_test.h`は以下の4種類が存在し、riscv-testsインストール時の`--prefix`で指定できる。

記号 | 説明 | URL
--- | --- | ---
`p`  | 仮想メモリ無し、コア1個、割込無し | <https://github.com/riscv/riscv-test-env/blob/81e58d0068858978c2415ae115144078434ea31d/p/riscv_test.h>
`pm` | 仮想メモリ無し、複数コア | <https://github.com/riscv/riscv-test-env/blob/81e58d0068858978c2415ae115144078434ea31d/pm/riscv_test.h>
`pt` | 仮想メモリ無し、タイマー割込有り | <https://github.com/riscv/riscv-test-env/blob/81e58d0068858978c2415ae115144078434ea31d/pt/riscv_test.h>
`pv` | 仮想メモリ有り | <https://github.com/riscv/riscv-test-env/blob/81e58d0068858978c2415ae115144078434ea31d/pv/riscv_test.h>

以下では仮想メモリ無し、コア1個、割込無しの`p/riscv_test.h`を考える。

`../rv64ui/add.S`はテストコードの本体であり、`rv64`版と共通化されている。
以下で詳細を見ていく。

### `isa/rv64ui/add.S`

<https://github.com/riscv-software-src/riscv-tests/blob/e65ecdf941a5484af27f9be223fb655ebcb0398b/isa/rv64ui/add.S>

```
#include "riscv_test.h"
#include "test_macros.h"

RVTEST_RV64U
RVTEST_CODE_BEGIN

(...略...)
  TEST_RR_OP( 2,  add, 0x00000000, 0x00000000, 0x00000000 );
  TEST_RR_OP( 3,  add, 0x00000002, 0x00000001, 0x00000001 );
  TEST_RR_OP( 4,  add, 0x0000000a, 0x00000003, 0x00000007 );
(...略...)
  TEST_PASSFAIL

RVTEST_CODE_END

  .data
RVTEST_DATA_BEGIN

  TEST_DATA

RVTEST_DATA_END
```

重要な部分だけ抜き出して

```
TEST_RR_OP( 4,  add, 0x0000000a, 0x00000003, 0x00000007 );
TEST_PASSFAIL
```

というコードについて考える。

* `TEST_RR_OP`マクロ: テストの実行
  - テスト番号 : `4`
  - テスト内容 : `0xa = 0x3 + 0x7`
* `TEST_PASSFAIL`マクロ: テスト結果の成否判定

これらを順番に見ていく。

# テストの実行処理

まずテストの実行部分を見ていく。

### `TEST_RR_OP`マクロ

<https://github.com/riscv-software-src/riscv-tests/blob/e65ecdf941a5484af27f9be223fb655ebcb0398b/isa/macros/scalar/test_macros.h#L124,L129>

```
#define TEST_RR_OP( testnum, inst, result, val1, val2 ) \
    TEST_CASE( testnum, x14, result, \
      li  x1, MASK_XLEN(val1); \
      li  x2, MASK_XLEN(val2); \
      inst x14, x1, x2; \
    )
```

これで`TEST_RR_OP`マクロの定義が分かった。つまり

```
TEST_RR_OP( 4,  add, 0x0000000a, 0x00000003, 0x00000007 );
```

は以下のように展開される。

```
TEST_CASE( 4, x14, 0x0000000a, \
  li  x1, MASK_XLEN(0x00000003);
  li  x2, MASK_XLEN(0x00000007);
  add x14, x1, x2;
)
```

さらに`TEST_CASE`の定義を見ていく。

### `TEST_CASE`マクロ

<https://github.com/riscv-software-src/riscv-tests/blob/e65ecdf941a5484af27f9be223fb655ebcb0398b/isa/macros/scalar/test_macros.h#L13,L18>

```
#define TEST_CASE( testnum, testreg, correctval, code... ) \
test_ ## testnum: \
    code; \
    li  x7, MASK_XLEN(correctval); \
    li  TESTNUM, testnum; \
    bne testreg, x7, fail;
```

さっき得られたコードにこのマクロ定義を適用すると、以下のアセンブリが得られる。

```
test_4:
  li  x1, MASK_XLEN(0x00000003)
  li  x2, MASK_XLEN(0x00000007)
  add x14, x1, x2
  li  x7, MASK_XLEN(0x0000000a)
  li  TESTNUM, 4
  bne x14, x7, fail
```

残りの不明箇所は

* 大文字の`TESTNUM`の定義 (実は`gp`レジスタ)
* `MASK_XLEN`マクロの定義 (実は`rv32ui`では意味を持たない)
* `fail`とは何か (実はテスト失敗時に実行されるコードのアドレス)

この三点を見ていく。

### `TESTNUM`レジスタ

<https://github.com/riscv/riscv-test-env/blob/0666378f353599d01fc48562b431b1dd049faab5/p/riscv_test.h#L243>

```
#define TESTNUM gp
```

つまり`TESTNUM`は`rv32ui`の`gp`レジスタとして展開される。

### `MASK_XLEN`マクロ

<https://github.com/riscv-software-src/riscv-tests/blob/e65ecdf941a5484af27f9be223fb655ebcb0398b/isa/macros/scalar/test_macros.h#L11>

```
#define MASK_XLEN(x) ((x) & ((1 << (__riscv_xlen - 1) << 1) - 1))
```

この`__riscv_xlen`はコンパイラによって自動で`32`か`64`に置換される。
今回は`rv32ui`用のコンパイラを使う想定なので、コンパイル時に32に置換される。
よって`MASK_XLEN`マクロは

```
#define MASK_XLEN(x) ((x) & ((1 << 31 << 1) - 1))
```

つまり`rv32`系のアーキテクチャでは

```
#define MASK_XLEN(x) (x & 0xFFFFFFFF)
```

のようになり、意味がなくなる。
これは`rv64`系アーキテクチャで

```
#define MASK_XLEN(x) (x & 0x00000000FFFFFFFF)
```

のようにして、レジスタ`x`の上位32bitをマスクするための処方箋といえる。

### `fail`のアドレス

これはテスト失敗時に実行するアドレスになる。詳細は次節を参照。

# テスト結果の成否判定

最後にテスト結果の評価部分を見ていく。

### テストを実行したか判定

<https://github.com/riscv-software-src/riscv-tests/blob/e30978a71921159aec38eeefd848fca4ed39a826/isa/macros/scalar/test_macros.h#L708,L713>

```
#define TEST_PASSFAIL \
        bne x0, TESTNUM, pass; \
fail: \
        RVTEST_FAIL; \
pass: \
        RVTEST_PASS \
```

この`fail`が、前述の`TEST_CASE`マクロに出てきたラベルで、`RV_TEST_FAIL`で定義される処理のアドレスを指している。
`TESTNUM`は前述の通り`gp`レジスタに置換される。この`gp`には実行したテストの個数が格納される。

つまりこのコードをざっくりまとめると

```
if (実行したテストの個数 != 0) {
  goto pass;
} else {
  goto fail;
}
```

以下では

* `RVTEST_PASS`
* `RVTEST_FAIL`

という2つの処理を見ていく。

### 成功時の処理

<https://github.com/riscv/riscv-test-env/blob/81e58d0068858978c2415ae115144078434ea31d/p/riscv_test.h#L237,L242>

```
#define RVTEST_PASS                                                     \
        fence;                                                          \
        li TESTNUM, 1;                                                  \
        li a7, 93;                                                      \
        li a0, 0;                                                       \
        ecall
```

`TESTNUM`は前述の通り`gp`レジスタに展開される。
よって以下のアセンブリコードが得られる。

```
fence
li gp, 1
li a7, 93
li a0, 0
ecall
```

* `gp`に1を入れているのは、テストが実行さたことを表す意図がある。1-31桁目に失敗したテスト番号入るが、今回は失敗していないので0で埋めている。
* `93`はプログラムを強制終了させるLinuxのシステムコール番号。`ecall`でLinux Kernelを呼んで強制終了させる意図がある。
* `a0`に0を入れているのは、プログラムの戻り値を正常にする意図がある。

### 失敗時の処理

<https://github.com/riscv/riscv-test-env/blob/81e58d0068858978c2415ae115144078434ea31d/p/riscv_test.h#L245,L252>

```
#define RVTEST_FAIL                                                     \
        fence;                                                          \
1:      beqz TESTNUM, 1b;                                               \
        sll TESTNUM, TESTNUM, 1;                                        \
        or TESTNUM, TESTNUM, 1;                                         \
        li a7, 93;                                                      \
        addi a0, TESTNUM, 0;                                            \
        ecall
```

* `TESTNUM`は前述の通り`gp`レジスタに展開される。
* `1:`はgnu assembler (gas)のlocal label機能で、ユニークなラベルに置換される
* `1b`は直近の`1:`のラベルのアドレスを指す

つまり

```
1:      beqz TESTNUM, 1b;
```

の行は、`gp`レジスタが0の場合に無限ループする。
これで実行したテストの個数が0でないことが保証される。

こうして以下のアセンブリコードを得る。

```
fence

hoge:
beqz gp, hoge

sll gp, gp, 1
or gp, gp, 1
li a7, 93
addi a0, gp, 0
ecall
```

* `sll`により、`gp`の1-31桁目に失敗したテストの番号を入れている。
* `or`により、`gp`のLSBに1を立てて、テストが実行されたことを表している。
* `93`は前述のLinuxシステムコール番号。
* `a0`に`gp`の値をコピーすることで、プログラムの戻り値に失敗したテストの番号を設定している。

# 最終的なアセンブリ出力

これまでの話を全てまとめると

```
TEST_RR_OP( 4,  add, 0x0000000a, 0x00000003, 0x00000007 );
TEST_PASSFAIL
```

というC言語のコードは、以下のアセンブリに変換される。

```
; ----- TEST_RR_OPの部分 -----

test_4:
  li  x1, (0x00000003 & 0xFFFFFFFF)
  li  x2, (0x00000007 & 0xFFFFFFFF)
  add x14, x1, x2
  li  x7, (0x0000000a & 0xFFFFFFFF)
  li  gp, 4
  bne x14, x7, fail


; ----- TEST_PASSFAILの部分 -----

bne x0, gp, pass

fail:
  fence

hoge:
  beqz gp, hoge
  sll gp, gp, 1
  or gp, gp, 1
  li a7, 93
  addi a0, gp, 0
  ecall

pass:
  fence
  li gp, 1
  li a7, 93
  li a0, 0
  ecall
```

なおリンカスクリプトは以下で与えられる。
通常のriscvのバイナリと同様に、開始アドレスが`0x80000000`になっている。

<https://github.com/riscv/riscv-test-env/blob/0666378f353599d01fc48562b431b1dd049faab5/p/link.ld>

# アセンブリを見やすくしたもの

テストの種類を追加し、レジスタ名の形式を統一し、定数を畳み込んで整理したコードの例。

```
; ---------- 各テストを実行 ----------

test_2:
  li  ra, 0x3        ; ra = 0
  li  sp, 0x7        ; sp = 0
  add a4, ra, sp     ; a4 = add(0,0)
  li  t2, 0xa        ; t2 = 0
  li  gp, 2          ; gp = 2 (テスト番号)
  bne a4, t2, fail   ; もし add(0,0) != 0 なら失敗時の処理に移る

test_3:
  li  ra, 0x3        ; ra = 1
  li  sp, 0x7        ; sp = 1
  add a4, ra, sp     ; a4 = add(1,1)
  li  t2, 0xa        ; t2 = 2
  li  gp, 3          ; gp = 3 (テスト番号)
  bne a4, t2, fail   ; もし add(1,1) != 2 なら失敗時の処理に移る

test_4:
  li  ra, 0x3        ; ra = 3
  li  sp, 0x7        ; sp = 7
  add a4, ra, sp     ; a4 = add(3,7)
  li  t2, 0xa        ; t2 = 11
  li  gp, 4          ; gp = 4 (テスト番号)
  bne a4, t2, fail   ; もし add(3,7) != 11 なら失敗時の処理に移る

; ---------- 実行したテストの個数判定 ----------

bne zero, gp, pass   ; if (実行したテストの個数 != 0) { 全テスト成功時の処理に移る }

; ---------- 失敗時 ----------

fail:
  fence

hoge:
  beqz gp, hoge      ; if (実行したテストの個数 == 0) { 無限ループ }
  sll gp, gp, 1      ; 失敗したテストの番号をgpの1--31桁目に入れる
  or gp, gp, 1       ; テスト実施のフラグを立てる
  li a7, 93          ; 93はプログラムを強制終了させるLinuxのシステムコール番号
  addi a0, gp, 0     ; プログラムの戻り値に、失敗したテストの番号を入れる
  ecall              ; Linuxのシステムコールを呼び出す

; ---------- 全テスト成功時 ----------

pass:                ; 全テスト成功時
  fence
  li gp, 1           ; テスト実施のフラグを立てる
  li a7, 93          ; 93はプログラムを強制終了させるLinuxのシステムコール番号
  li a0, 0           ; プログラムの戻り値を正常終了にする
  ecall              ; Linuxのシステムコールを呼び出す
```
