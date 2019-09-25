## 13. ベクトル固定小数点演算命令

固定小数点算術演算をサポートするための命令が定義されている。

Nビットの各ベクトル要素には、-2^N-1から2^N-1-1までの符号付整数か、0から2^N-1までの符号なし整数を格納することができる。固定小数点命令はスケーリングと丸めをサポートすることにより、幅の狭いオペランドでも精度を保ちながら演算することを助け、計算結果のフォーマットの範囲から値が飽和したことによるオーバフローを適切に処理することができるようになる。

> 上記で説明したビット幅を拡張する整数命令はオーバフローが発生する抑制するためにも使用できる。

### 13.1. 単一ビット幅ベクトル飽和加算減算命令

飽和を行う整数加減算命令を定義する。どちらも符号付と符号なしの形式をサポートする。計算結果がオーバフローした場合、結果はその近傍で表現できる値に変換され、`vxsat`ビットが設定される。

```
# 符号なし整数飽和加算命令
vsaddu.vv vd, vs2, vs1, vm   # ベクトル - ベクトル
vsaddu.vx vd, vs2, rs1, vm   # ベクトル - スカラ
vsaddu.vi vd, vs2, imm, vm   # ベクトル - 即値

# 符号付整数飽和加算命令
vsadd.vv vd, vs2, vs1, vm   # ベクトル - ベクトル
vsadd.vx vd, vs2, rs1, vm   # ベクトル - スカラ
vsadd.vi vd, vs2, imm, vm   # ベクトル - 即値

# 符号なし整数飽和減算命令
vssubu.vv vd, vs2, vs1, vm   # ベクトル - ベクトル
vssubu.vx vd, vs2, rs1, vm   # ベクトル - スカラ

# 符号付整数飽和減算命令
vssub.vv vd, vs2, vs1, vm   # ベクトル - ベクトル
vssub.vx vd, vs2, rs1, vm   # ベクトル - スカラ
```

### 13.2. 単一幅ベクトル平均加算減算命令

平均加算減算命令は`vxrm`の設定に基づき、2つの値の加減算の結果と1ビットを加算したものを1ビット右にシフトしたものを返す。この命令ではオーバフローは発生しない。

```
# vrxm=rnu の場合、 round = 1

# 平均加算
# result = (src1 + src2 + round) >> 1;

# 整数平均加算命令
vaadd.vv vd, vs2, vs1, vm   # ベクトル - ベクトル
vaadd.vx vd, vs2, rs1, vm   # ベクトル - スカラ
vaadd.vi vd, vs2, imm, vm   # ベクトル - 即値

# 平均減算
# result = (src1 - src2 + round) >> 1;

# 整数平均減算命令
vasub.vv vd, vs2, vs1, vm   # ベクトル - ベクトル
vasub.vx vd, vs2, rs1, vm   # ベクトル - スカラ
```

### 13.3. 丸めと飽和付き 単一ビット幅ベクトル分数乗算命令

符号付整数分数乗算命令は2つのSEWビット長の入力から2*SEWの積を計算し、結果をSEW-1ビット右シフトし、`vxrm`の設定に基づいて丸め処理を行い、SEWビットに結果を飽和する。計算結果に飽和が発生すると、`vxsat`ビットが設定される。

```
# 丸めと飽和付き 符号付き分数乗算命令
vsmul.vv vd, vs2, vs1, vm  # vd[i] = clip((vs2[i]*vs1[i]+round)>>(SEW-1))
vsmul.vx vd, vs2, rs1, vm  # vd[i] = clip((vs2[i]*x[rs1]+round)>>(SEW-1))
```

> Nビットの符号付整数を乗算したとき、最大値は-2^N-1*-2^N-1を計算したときの+2^{2N}-2であり、2Nビットのうちこれは符号ビット(実際は0)は1ビットである。これ以外の乗算では、符号ビットは結果の2Nビット中で2ビットである。Nビットの結果のなかでより精度を上げるために、積を右にN-1シフトし、最大の積の結果を飽和させる代わりに他の計算結果の精度を向上させる。

> より固定小数点命令のサポートを強化するために、`vxrm`で制御される丸めモードを`vmulhu`, `vmulhsu, `vmulh`に追加することも考えられる。これらの命令では飽和は発生しない。
>
> 

### 13.4. ビット幅拡張飽和付き累積乗算加算命令

ビット幅拡張飽和付き乗算加算命令はSEWビット同士の入力値を乗算し2SEWビットの積を生成する。この結果はSEW/2ビットだけ右にシフトされ`vxrm`のビット設定に基づいて丸めが行われ、2SEWビットの書き込みレジスタアキュムレータの値と加算される。結果が書き込みレジスタの中で飽和すると、オーバフローが発生し、`vxsat`ビットが設定される。

もし乗算オペランドが符号付であれば、乗算の結果は符号付の値としてオーバフロー・飽和処理が行われる。両方の乗算オペランドが符号なしであれば、乗算の結果は符号なしの値としてオーバフロー・飽和処理が行われる。

| SEW  | 除算のビット幅 | 丸めのビット幅 | アキュムレータのビット幅 | ガードビット |
| :--- | :------------- | :------------- | :----------------------- | :----------- |
| 8    | 16             | 12             | 16                       | 4            |
| 16   | 32             | 24             | 32                       | 8            |
| 32   | 64             | 48             | 64                       | 16           |

```
# ビット幅拡張符号なし累積乗算加算命令
vwsmaccu.vv vd, vs1, vs2, vm # vd[i] = clipu((+(vs1[i]*vs2[i]+round)>>SEW/2)+vd[i])
vwsmaccu.vx vd, rs1, vs2, vm # vd[i] = clipu((+(x[rs1]*vs2[i]+round)>>SEW/2)+vd[i])

# ビット幅拡張符号付き累積乗算加算命令
vwsmacc.vv vd, vs1, vs2, vm  # vd[i] = clip((+(vs1[i]*vs2[i]+round)>>SEW/2)+vd[i])
vwsmacc.vx vd, rs1, vs2, vm  # vd[i] = clip((+(x[rs1]*vs2[i]+round)>>SEW/2)+vd[i])

# ビット幅拡張符号付きー符号なし累積乗算加算命令
vwsmaccsu.vv vd, vs1, vs2, vm
             # vd[i] = clip(((signed(vs1[i])*unsigned(vs2[i])+round)>>SEW/2)+vd[i])
vwsmaccsu.vx vd, rs1, vs2, vm
             # vd[i] = clip(((signed(x[rs1])*unsigned(vs2[i])+round)>>SEW/2)+vd[i])

# ビット幅拡張符号なしー符号付き累積乗算加算命令
vwsmaccus.vx vd, rs1, vs2, vm
             # vd[i] = clip(((unsigned(x[rs1])*signed(vs2[i])+round)>>SEW/2)+vd[i])

# xvrm=rnuの場合、round = (1 << (SEW/2-1))
```

> より柔軟なスケーリング・シフト量をサポートするためには、4番目のソースオペランドが必要となってしまう。

### 13.5. 単一幅ベクトルスケーリングシフト命令

これらの命令は入力値を右シフトし、`vxrm`の設定に基づいてシフトした値を丸める。スケーリング右シフト命令はゼロ拡張(`vssrl`)と符号拡張(`vssra`)の形式を用意している。ベクトル・スカラのシフト量の下位のlg2(SEW)ビットが使用される。即値もサポートされるが、シフト量は最大でも31までである。

```
 # vxrm=rnuでは、round = 1 << (src2-1)である。ここでsrc2 は vs1[i], x[rs1], uimm のどれか。
 # スケーリング論理右シフト命令
 vssrl.vv vd, vs2, vs1, vm   # vd[i] = ((vs2[i] + round)>>vs1[i])
 vssrl.vx vd, vs2, rs1, vm   # vd[i] = ((vs2[i] + round)>>x[rs1])
 vssrl.vi vd, vs2, uimm, vm   # vd[i] = ((vs2[i] + round)>>uimm)

 # スケーリング算術右シフト命令
 vssra.vv vd, vs2, vs1, vm   # vd[i] = ((vs2[i] + round)>>vs1[i])
 vssra.vx vd, vs2, rs1, vm   # vd[i] = ((vs2[i] + round)>>x[rs1])
 vssra.vi vd, vs2, uimm, vm   # vd[i] = ((vs2[i] + round)>>uimm)
```

### 13.6. ビット幅縮小固定小数点ベクトルクリップ命令

`vnclip`命令は固定小数点の値をよりビット幅の小さいレジスタに書き込む。この命令では書き込みレジスタのフォーマットに合わせて丸め、スケーリング、飽和をサポートする。

命令の2番目の引数(ベクトル要素、スカラ値、即値のどれか)は本命令の右シフト量を示し、これによりスケーリングが行われる。ベクトルレジスタ要素、スカラシフト量のうちlg2(2*SEW)ビットが使用される(例えば、SEW=64ビットで、32ビットにビット幅縮小するのであれば、シフト量は6ビット分使用される)。即値形式の場合では、最大で31までがサポートされる。

```
 # 符号なしビット幅縮小クリップ命令
 vnclipu.vv vd, vs2, vs1, vm   # ベクトル - ベクトル
 vnclipu.vx vd, vs2, rs1, vm   # ベクトル - スカラ
 vnclipu.vi vd, vs2, uimm, vm   # ベクトル - 即値

# 符号付きビット幅縮小クリップ命令,  vd[i] = clip(round(vs2[i] + rnd) >> vs1[i])
#                              SEW           2*SEW                 SEW
 vnclip.vv vd, vs2, vs1, vm   # ベクトル - ベクトル
 vnclip.vx vd, vs2, rs1, vm   # ベクトル - スカラ
 vnclip.vi vd, vs2, uimm, vm   # ベクトル - 即値
```

`vnclipu`,/`vnclip`命令では、丸めモードは`vxrm`CSRに基づいて実行される。丸めは飽和処理実行前に、書き込み値の最下位ビットに対して行われる。

`vnclipu`命令では、シフトして丸められたソース値は符号なしの値として扱われ、値がオーバフローした場合には符号なしの整数として飽和処理が行われる。

`vnclip`命令では、シフトして丸められた値は符号付の値として扱われ、値がオーバフローした場合は符号付きの整数として飽和処理が行われる。

書き込みレジスタのどれかの要素で飽和が発生すると、`vxsat`レジスタの`vxsat`ビットが設定される。