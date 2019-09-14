## 8. ベクトルAMO操作(`Zvamo`)

> Profiles will dictate whether vector AMO operations are supported. The expectation is that the Unix profile will require vector AMO operations.

> プロファイルでは、どこにベクトルAMO命令がサポートされているかについて述べている。想定では、UnixのプロファイルではベクトルAMO操作が必要になる。

ベクトルAMO命令がサポートされている場合、スカラのZaamo命令(標準A拡張からのアトミック操作)のサポートは必須である。

ベクトルAMO操作は標準的なAMOメジャーオペコードに基づく、不必要なビット幅のエンコーディングを使用している。アックアクティブ要素はアトミックなRead-modify-writeを単一のメモリ領域に対して実行する。

```
AMOメジャーオペコードを使用したベクトルAMO命令のフォーマット

31    27 26  25  24      20 19       15 14   12 11      7 6     0
 amoop  |wd| vm |   vs2    |    rs1    | width | vs3/vd  |0101111| VAMO*
   5      1   1      5           5         3        5        7
vs2[4:0] アドレスを保持しているVレジスタ
vs3/vd[4:0] 読み込みオペランドと書き込みオペランドを保持しているVレジスタ。

vmはベクトルマスクを表現している。
width[2:0] メモリ要素のサイズを示している。これはスカラAMOのものとは異なる。
amoop[4:0] AMO操作を示す。
wd は元のメモリの阿多いをvdに書き込むかを指定する(1=書き込む、0=書き込まない)
```

AMOはインデックス操作と同様のアドレッシングモードを持っているが、即値のオフセットは持たない。`vs2`はバイトオフセットを保持するベクトルであり、スカラのベースレジスタ`rs1`とオフセットの値を加算してAMO操作を行うアドレスを決める。

`vs2`ベクトルレジスタは各要素のバイトオフセットを保持しており、`vs3`はアトミックメモリ操作を実行する値を保持しているベクトルレジスタである。

If the `wd` bit is set, the `vd` register is written with the initial value of the memory element. If the `wd` bit is clear, the `vd` register is not written.

`wd`ビットが設定されている場合、`vd`レジスタにメモリ要素のアトミック操作前の値が書き込まれる。もし`wd`がクリアされている場合、`vd`レジスタには何も書きこまれない。

> `wd`がクリアされている場合、メモリシステムはメモリの初期値を返す必要がなく、また`vd`の初期値は保持される。

> AMO命令は、レジスタリネーミングにおいて全体的なメモリパイプラインの読み込みポートを削減するように、読み込みデータのレジスタを上書きするように設計されている。これは、ベクトルインデックス操作と同じアドレッシングモードをサポートするため、またベクトルAMOは通常の並列メモリリダクション操作において結果をレジスタに保持する必要がないためである。

ベクトルAMO操作は、`aq`と`rl`ビットはゼロが設定されているものとして動作し、同じhart乗の他の命令との順序を守るように設計されている。

ベクトルAMOは、同じ命令内において要素に対する演算の順序は保証しない。

|                     | Width [2:0] |      |      | メモリのビット | レジスタのビット | Opcode   |
| :------------------ | :---------- | :--- | :--- | :------------- | ---------------- | -------- |
| 標準的なスカラのAMO | 0           | 1    | 0    | 32             | XLEN             | AMO*.W   |
| 標準的なスカラのAMO | 0           | 1    | 1    | 64             | XLEN             | AMO*.D   |
| 標準的なスカラのAMO | 1           | 0    | 0    | 128            | XLEN             | AMO*.Q   |
| ベクトルAMO         | 1           | 1    | 0    | 32             | vl*SEW           | VAMO*W.V |
| ベクトルAMO         | 1           | 1    | 1    | 64             | vl*SEW           | VAMO*E.V |

メモリのビットは、メモリ中にアクセスされる要素のサイズである。

レジスタのビットは、レジスタ中の要素のサイズである。

ベクトルAMOには2種類のサイズの命令がが存在している。1つは32ビットを1ワードとし、もう一つhあSEWビットを1ワードとするものである。32ビットベクトルAMO操作では、SEWは少なくとも32ビット以上でなければならず、そうでなければ不正命令例外が発生する。SEW > 32ビットであれば、メモリから戻ってくる値はSEW長にするために符号拡張が行われる。

SEWがXLENよりも小さい場合、ベクトル`vs2`内のアドレスはXLENビットまで符号拡張される。SEWがXLEｎよりも大きい場合、不正命令例外が発生する。

ベクトルAMO命令はメモリ要素の幅はスカラアーキテクチャの実装でサポートされている幅のみをサポートしている。他のデータサイズでは、不正命令例外が発生する。

The vector `amoop[4:0]` field uses the same encoding as the scalar 5-bit AMO instruction field, except that LR and SC are not supported.

ベクトルお`amoop[4:0]`フィールドはスカラAMO命令フィールドの5ビットと同一であるが、LR/SCをサポートしていない点が異なる。

| amoop |      |      |      |      | opcode   |
| :---- | :--- | ---- | ---- | ---- | -------- |
| 0     | 0    | 0    | 0    | 1    | vamoswap |
| 0     | 0    | 0    | 0    | 0    | vamoadd  |
| 0     | 0    | 1    | 0    | 0    | vamoxor  |
| 0     | 1    | 1    | 0    | 0    | vamoand  |
| 0     | 1    | 0    | 0    | 0    | vamoor   |
| 1     | 0    | 0    | 0    | 0    | vamomin  |
| 1     | 0    | 1    | 0    | 0    | vamomax  |
| 1     | 1    | 0    | 0    | 0    | vamominu |
| 1     | 1    | 1    | 0    | 0    | vamomaxu |

アセンブリの文法では書き込みレジスタとして`x0`を指定することで、戻り値が不要であることを示すことができる(`wd=0`)。

```
# 32ビットベクトルAMOs
vamoswapw.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamoswapw.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamoaddw.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamoaddw.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamoxorw.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamoxorw.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamoandw.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamoandw.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamoorw.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamoorw.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamominw.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamominw.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamomaxw.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamomaxw.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamominuw.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamominuw.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamomaxuw.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamomaxuw.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

# SEWビット ベクトル AMOs
vamoswape.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamoswape.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamoadde.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamoadde.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamoxore.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamoxore.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamoande.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamoande.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamoore.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamoore.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamomine.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamomine.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamomaxe.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamomaxe.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamominue.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamominue.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。

vamomaxue.v vd, (rs1), v2, vd,  v0.t # wd=1であるため変更前の値をメモリに書き込む。
vamomaxue.v x0, (rs1), v2, vs3, v0.t # wd=0であるため変更前の値をメモリに書き込まない。
```




## 9. ベクトルメモリのアライメント制約

ベクトルメモリ命令によってアクセスされる要素が、メモリアクセス際に対してアライメントしていない場合、アドレスミスアライン例外を発生させることもできるし、正しく要素を処理することもできる。

ベクトルメモリアクセスの制約はスカラメモリアクセスアトミック命令の制約と同様である。

## 10. ベクトルメモリのコンシステンシモデル

ベクトルメモリ命令は、ローカルのHartのプログラムの順番に従って実行される。ベクトルメモリ命令は命令レベルでRVWMOのメモリコンシステンシモデルに従い、要素の操作は構文的に独立したスカラー命令の要素順に並べられたシーケンスによって実行されるかのように、命令内で順序付けられる。ベクトルインデックスオーダーのストア命令はメモリに対して要素の順序を守って書き込みが行われる。ベクトルインデックスのアンオーダーストア命令は、単一の命令の中で要素の順序を守らない。

> もう少し詳細を肉付けする必要あり。
