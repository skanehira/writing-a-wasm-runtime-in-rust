---
Wasmバイナリの構造
---

Wasm RuntimeがWasmバイナリを実行する際は、大きく分けてWasmバイナリのデコードと命令処理の2ステップがある。
バイナリのデコードをする上でバイナリの構造を知る必要があるため、本章ではその構造について解説していく。

本章を読み終えるころには、バイナリの構造がどのような形になっているのかを知ることができる。

## Wasmバイナリの全体像
Wasmバイナリは先頭に8バイトのプリアンブル、それに続いて各種セクションという構造となっている。

プリアンブルは`Magic Number`の`'\0asm'`とバージョンの`1`の値で構成されていて、それぞれ4バイトずつファイルの先頭から配置されている。

以下が`wat2wasm -v`で出力されたバイナリの内容を加工して、少し説明を追加したものである。

```
           \0asm
         ┌───┴───┐
0000000: 0061 736d      ; WASM_BINARY_MAGIC
~~~~~~~  ~~             ~~~~~~~~~~~~~~~~~~~~ 
 │        │                   │
 │        │                   └ コメント
 │        └ 16進数表記、2桁で1バイト
 └ アドレスのオフセット

0000004: 0100 0000      ; WASM_BINARY_VERSION
```

以降、バイナリ構造について説明するときは、基本的にこの出力を少しわかりやすくした内容をもとに解説していく。

セクションは複数個あり、各セクションに実行時に必要な情報が保存されている。
たとえば関数のシグネチャ情報やメモリ初期化の情報、実行する命令の情報などがある。

ちなみに、セクションは任意のため、プリアンブルのみから構成される最小限のWasmバイナリを作ることもできる。

本書では次のセクションを実装していくので、それ以外のセクションに関しては[仕様書](https://www.w3.org/TR/wasm-core-1/#sections%E2%91%A0)を参照してほしい。

| セクション         | 概要                                 |
|--------------------|--------------------------------------|
| `Type Section`     | 関数シグネチャの情報                 |
| `Code Section`     | 関数ごとの命令などの情報             |
| `Function Section` | 関数シグネチャへの参照情報           |
| `Memory Section`   | 線形メモリの情報                     |
| `Data Section`     | 初期化時にメモリに配置するデータ情報 |
| `Export Section`   | 他モジュールへのエクスポート情報     |
| `Import Section`   | 他モジュールのインポート情報         |

以降は各セクションのデータ構造について説明していく。

### [Type Section](https://www.w3.org/TR/wasm-core-1/#type-section%E2%91%A0)

関数シグネチャ情報を保持する領域である。
シグネチャは簡潔にいうと関数の型のことである。

関数シグネチャは次の組み合わせで一意に決まる。

- 引数の型と個数、並び
- 戻り値の型と個数、並び

たとえばリスト1の関数`$a`と`$b`は引数の並びや戻り値の有無といった差異があるのでシグネチャは違うが、`$a`と`$c`は同じシグネチャとなる。
シグネチャはいわば関数の入出力の定義なので、関数の中身には依存しないため、`$a`と`$c`は同じシグネチャ情報を参照することになる。

```wat:リスト1
(module
  (func $a (param i32 i64))
  (func $b (param i64 i32) (result i32 i64)
    (local.get 1)
    (local.get 0)
  )
  (func $c (param i32 i64))
)
```

関数シグネチャは関数を実行するとき、引数と戻り値を何個スタックに積むかを知るための情報である。
具体的にシグネチャ情報がどのように使用されるのかは`Runtime`実装の章で詳しく解説する。

リスト2がバイナリ構造である。

```:リスト2
; section "Type" (1)
0000008: 01       ; section code
0000009: 0d       ; section size
000000a: 02       ; num types
; func type 0
000000b: 60       ; func       
000000c: 02       ; num params 
000000d: 7f       ; i32        
000000e: 7e       ; i64        
000000f: 00       ; num results
; func type 1
0000010: 60       ; func       
0000011: 02       ; num params 
0000012: 7e       ; i64        
0000013: 7f       ; i32        
0000014: 02       ; num results
0000015: 7f       ; i32        
0000016: 7e       ; i64        
```

先頭2バイトはそれぞれ`section code`と`section size`となっていて、これはすべてのセクションで共通である。
正式な名称はないが、本書ではセクションヘッダーと呼ぶことにする。

```
; section "Type" (1)
0000008: 01          ; section code
0000009: 0d          ; section size
000000a: 02          ; num types
```

`section code`はセクションを識別するための一意の値で、`Type Section`の場合は`1`である。

`section size`は先頭2バイトを除いたセクションデータのバイト数である。
これにより、セクションをデコードする際はどこまでバイナリを読み取ればいいのか分かる。

`num types`は関数シグネチャの数である。この数の分だけ、関数シグネチャをデコードしていく。

セクションの残りの部分が関数シグネチャの定義となっている。
各関数シグネチャは`0x60`から始まり、引数の個数と型、戻り値の個数と型という順で定義されている。

リスト3では`func type 0`は`(func $a (param i32 i64))`と`(func $c (param i32 i64))`、
`func type 1`は`(func $b (param i64 i32) (result i32 i64))`のシグネチャ情報となっている。

```:リスト3
; func type 0
000000b: 60    ; func         ┐                              
000000c: 02    ; num params   │ (func $a (param i32 i64)) 
000000d: 7f    ; i32          ├ (func $c (param i32 i64)) 
000000e: 7e    ; i64          │                           
000000f: 00    ; num results  ┘                           
; func type 1
0000010: 60    ; func         ┐                     
0000011: 02    ; num params   │                     
0000012: 7e    ; i64          │ (func $b            
0000013: 7f    ; i32          ├   (param i64 i32)   
0000014: 02    ; num results  │   (result i32 i64)  
0000015: 7f    ; i32          │ )                   
0000016: 7e    ; i64          ┘                     
```

関数シグネチャのデコードはおおよそ次のことを行う。

1. 1バイト読み取って、`0x60`であるかを確認
2. 1バイト読み取って、引数の個数を取得
3. 2で取得した個数の分だけバイトを読み取る  
   たとえば2なら2バイト読み取る
4. 3で読み取ったバイト列を1バイトずつ読み進め、値に対応した型情報（`0x7e`ならi64型）を取得
5. 2〜4と同様の手順で戻り値の型情報を取得

### [Code Section](https://www.w3.org/TR/wasm-core-1/#code-section%E2%91%A0)

`Code Section`は主に関数の命令情報が保存されている領域である。

リスト4が`Code Section`のバイナリ構造である。

```:リスト4
; section "Code" (10)
000001d: 0a           ; section code
000001e: 0e           ; section size
000001f: 03           ; num functions
; function body 0
0000020: 02           ; func body size
0000021: 00           ; local decl count
0000022: 0b           ; end
; function body 1
0000023: 06           ; func body size
0000024: 00           ; local decl count
0000025: 20           ; local.get
0000026: 01           ; local index
0000027: 20           ; local.get
0000028: 00           ; local index
0000029: 0b           ; end
; function body 2
000002a: 02           ; func body size
000002b: 00           ; local decl count
000002c: 0b           ; end
```

`num functions`は関数の個数となっていて、この個数の数だけ関数デコードしていく。

残りの部分は各関数のローカル変数の定義と命令情報になっていて、これらを繰り返しデコードしていく。

`func body size`は関数本体のバイト数を示している。

`local decl count`はローカル変数の個数を示している。
`0`の場合は何もしないが、`1`以上の場合は後続のバイト列はローカル変数の型が定義される。

残りの`end`までのバイト列は関数の命令となっていて、`Runtime`はこれらの命令を処理していくことになる。

関数のデコードはおおよそ次のことを行う。

1. 1バイト読み取って、関数のサイズを取得
2. 1で取得した関数のサイズ分、バイト列を読み取る
3. 1バイト読み取って、ローカル変数の個数を取得
4. 3で取得した個数の数だけ、1バイトずつ読み取って型情報を取得
5. 2で読み取ったバイト列がなくなるまで、命令を取得

### [Function Section](https://www.w3.org/TR/wasm-core-1/#function-section%E2%91%A0)

`Function Section`は関数本体（`Code Section`）と型情報（`Type Section`）を紐付けるための情報を保持している領域である。

リスト5がバイナリ構造である。

```:リスト5
; section "Function" (3)
0000017: 03              ; section code
0000018: 04              ; section size
0000019: 03              ; num functions
000001a: 00              ; function 0 signature index
000001b: 01              ; function 1 signature index
000001c: 00              ; function 2 signature index
```

`function x signature index`の値が関数シグネチャへのインデックス情報（0-based）となっている。
たとえば`function 2`は`Type Section`の`0`のシグネチャをもつということになる。

少し分かりづらいので、関連性を図1にすると次のようになる。

![](./images/relation_type_func_body.png)

*図1*

### [Memory Section](https://www.w3.org/TR/wasm-core-1/#memory-section%E2%91%A0)

`Memory Section`は`Runtime`のメモリをどれくらい確保するかの情報が保存されている領域である。

メモリはページ単位で拡張できて、1ページ64KiBと[仕様](https://www.w3.org/TR/wasm-core-1/#memory-instances%E2%91%A0)で定められている。

メモリはリスト6のように`(memory $initial $max)`という書式となっていて、`2`は初期のメモリページ数、`3`は上限ページ数となっている。
`max`はオプショナルで、指定しない場合は上限はなし。

```:リスト6
(module
  (memory 2 3)
)
```

バイナリ構造はリスト7のようになっている。

```:リスト7
; section "Memory" (5)
0000008: 05             ; section code
0000009: 04             ; section size
000000a: 01             ; num memories
; memory 0
000000b: 01             ; limits: flags
000000c: 02             ; limits: initial
000000d: 03             ; limits: max
```

`num memories`はメモリの個数だが、version 1の仕様ではメモリは1モジュールに1つしか定義できないので、実質的にこの値は1で固定される。

`limits: flags`は`max`があるかどうかの判定のために使用する値で、`0`の場合は`initial`のみ、`1`の場合は`initial`と`max`があるということを意味している。
これにより、どのようにデコードすればいいのかが分かるようになる。

### [Data Section](https://www.w3.org/TR/wasm-core-1/#data-section%E2%91%A0)

`Data Section`は`Runtime`でメモリを確保したあとに配置するデータが定義されている領域である。
つまりメモリの初期データの定義である。

リスト8はメモリに`Hello, World!\n`文字列を定義している例である。

```:リスト8
(module
  (memory 1)
  (data 0 (i32.const 0) "Hello, World!\n")
)
```

データは`(data $memory $offset $data)`という書式となっていて、次の要素で構成されている。

- `$memory`はデータの配置先メモリのインデックス
- `$offset`はデータを配置するメモリのoffsetを算出するための命令列
- `$data`が実際にメモリに配置されるデータ

今回の例では、0番目のメモリの0バイト目から`Hello, World!\n`という文字列が配置される。

バイナリ構造はリスト9のようになっている。

```:リスト9
; section "Data" (11)
000000d: 0b                                   ; section code
000000e: 14                                   ; section size
000000f: 01                                   ; num data segments
; data segment header 0
0000010: 00                                   ; segment flags
0000011: 41                                   ; i32.const
0000012: 00                                   ; i32 literal
0000013: 0b                                   ; end
0000014: 0e                                   ; data segment size
; data segment data 0
0000015: 4865 6c6c 6f2c 2057 6f72 6c64 210a   ; data segment data
```

データは`segment`という単位となっていて、`segment`は複数存在することがある。
`segment`は`header`と`data`の領域から構成されていて、`header`は`offset`を算出する命令列、`data`は実際のデータを保持している。

`num data segments`は`segment`の個数である。

`data segment header`はデータ配置先のメモリやoffsetなどのメタデータを持っている領域となっている。これが`segment`の個数分ある。

`segment flags`はデータを配置するメモリのインデックスである。version 1ではメモリは1つしか定義できないので、実質0で固定される。

`i32.const`から`end`まではoffsetを算出するための命令列となっている。今回は固定値のみを扱うが、グローバルの値を参照もできる。

`data segment size`は実際に配置するデータの長さで、`data segment data`は実際に配置するデータそのものである。

図2はリスト9の`segment`の構造を表したもの。

![](./images/data_section.png)

*図2*

### [Export Section](https://www.w3.org/TR/wasm-core-1/#export-section%E2%91%A0)

`Export Section`は他モジュールからアクセスできる情報が定義されている領域である。
version 1ではメモリや関数などをエクスポートできる。

`Runtime`側は基本的にエクスポートした関数しか呼ぶことができないため、たとえば足し算をする関数を`Runtime`側から呼ぶ場合は関数をエクスポートする必要がある。

リスト10はモジュール自身が持っている関数`$dummy`を`dummy`という名前でエクスポートする例である。

```:リスト10
(module
  (func $dummy)
  (export "dummy" (func $dummy))
)
```

エクスポートの書式は`(export $name ($type $index))`となっている。
`$name`はエクスポートする名前で、`$type`は`func`や`memory`といったエクスポートするデータの種類となっている。
`$index`はそのデータのインデックスまたは名前である。たとえば`func 0`の場合は0番目の関数を指す。
今回の例では関数名`$dummy`を指定しているがバイナリになった時点ではインデックスに変換される。

バイナリ構造はリスト11のようになっている。

```:リスト11
; section "Export" (7)
0000012: 07                   ; section code
0000013: 09                   ; section size
0000014: 01                   ; num exports
0000015: 05                   ; string length
0000016: 6475 6d6d 79         ; export name (dummy)
000001b: 00                   ; export kind
000001c: 00                   ; export func index
```

`num exports`はエクスポートするデータ個数である。

`string length`はエクスポートする名前のバイト列の長さで、`export name`が実際の文字バイト列である。

`export kind`はデータの種類で、メモリの場合は`0x02`である。

`export func index`はエクスポートする関数のインデックスである。

### [Import Section](https://www.w3.org/TR/wasm-core-1/#import-section%E2%91%A0)

`Import Section`がモジュール外にあるメモリや関数などをインポートするための情報が定義されている領域である。
モジュール外とは、他モジュールや`Runtime`が用意したメモリや関数のことを指す。

今回はWASIを実装していくが、WASIの実体は`Runtime`側で実装された関数なので、それをインポートして使用する予定である。

リスト12は`adder`というモジュールから`add`という関数をインポートする例である。

```:リスト12
(module
  (import "adder" "add" (func (param i32 i32) (result i32)))
)
```

インポートの書式は`(import $module $name $type)`となっている。

`$module`はモジュール名で`$name`はインポートする関数名やメモリ名となる。
`$type`は型の定義情報となっていて、関数の場合は関数のシグネチャ情報、メモリの場合はメモリの`min`と`max`情報が定義される。

バイナリ構造はリスト13のようになっている。

```:リスト13
; section "Type" (1)
0000008: 01                ; section code
0000009: 07                ; section size
000000a: 01                ; num types
; func type 0
000000b: 60                ; func
000000c: 02                ; num params
000000d: 7f                ; i32
000000e: 7f                ; i32
000000f: 01                ; num results
0000010: 7f                ; i32
; section "Import" (2)
0000011: 02                ; section code
0000012: 0d                ; section size
0000013: 01                ; num imports
; import header 0
0000014: 05                ; string length
0000015: 6164 6465 72      ; import module name (adder)
000001a: 03                ; string length
000001b: 6164 64           ; import field name (add)
000001e: 00                ; import kind
000001f: 00                ; import signature index
```

`string length`は文字のバイト列の長さで、`import module name`は実際のモジュール名のバイト列となっている。

`import field name`はインポートする関数やメモリ名のバイト列となっている。

`import kind`はインポートの種類で、関数の場合は`0`である。

`import signature index`は関数のシグネチャ情報のインデックスで、`Type Section`の`func type 0`を指している。

## まとめ
本章では実装対象のセクションについて解説をした。
バイナリを扱ったことがない方は大変だと思うが、慣れるまで繰り返し本章を読み直してみてほしい。

次章では実際にWasmバイナリを実際にデコードする処理を実装していく。
