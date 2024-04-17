---
title: 浮動小数点数の min / max
tags:
  - IEEE754
  - 浮動小数点数
private: false
updated_at: '2020-05-28T18:03:11+09:00'
id: 41a09bd40acfceec6ec8
organization_url_name: null
slide: false
ignorePublish: false
---
多くのプログラミング言語には、「2つ以上の数値が与えられた時、その最小値あるいは最大値」を返す関数 (min / max) が用意されている。入力が整数や有理数であれば難しい話はないのだが、対象が浮動小数点数の場合は厄介な問題が起こる。具体的には、「NaN の扱い」と「0 の符号の扱い」だ。

浮動小数点数の NaN は、皆さんご存知の通り、順序付けられない。NaN が絡む場合の min / max 演算については、「入力に NaN が含まれていたら結果も NaN とする」「NaN を入力の欠落として扱い、NaN でない入力があればそれを返す」などの立場が考えられる。

もっと細かいことを言うと、NaN を返す場合に入力で与えられた NaN を返すか、正規化された NaN を返すかという違いもありうるし、signaling NaN の扱いも議論の余地があるかもしれないが、この記事では細かいことは扱わない。

0 の符号に関しては、 -0 < +0 と扱う、つまり

```c
min(-0, +0) = min(+0, -0) = -0
max(-0, +0) = max(+0, -0) = +0
```

とするのが最も直感的だ。ただ、プログラミング環境によってはそうなっていない場合がある。

IEEE 754 では min / max の他に min magnitude / max magnitude という演算が規定されており、プログラミング言語や命令セットによってはそれらに対応した命令を持っていることもあるが、この記事では扱わない。

# IEEE 754

言わずと知れた、浮動小数点数の規格が IEEE 754 である。IEEE 754 では浮動小数点数に関する各種演算を規定しているが、min / max がどう規定されているか確認してみよう。

## IEEE 754-1985

IEEE 754 の当初の版では、 min / max 演算は規定されていなかった。

## IEEE 754-2008

IEEE 754-2008 では、 `minNum` と `maxNum` が規定された。

```
sourceFormat minNum(source, source)
sourceFormat maxNum(source, source)
```

これらは、片方が quiet NaN でもう片方が通常の数であれば、通常の数の方を返す。つまり、quiet NaN を「入力の欠落」として扱う。一方、入力に signaling NaN が含まれる場合は quiet NaN を返す。

しかし、 0 の符号の扱いについては明言されていない。また、別の問題点として、 `minNum` と `maxNum` は signaling NaN の扱いについて非結合的となっている。

## IEEE 754-2019

IEEE 754-2019 では `minNum` / `maxNum` の代わりに `minimum` / `minimumNumber` / `maximum` / `maximumNumber` の4つが規定されている。

```
sourceFormat minimum(source, source)
sourceFormat minimumNumber(source, source)
sourceFormat maximum(source, source)
sourceFormat maximumNumber(source, source)
```

`minimum` と `maximum` は、片方が NaN なら quiet NaN を返す。つまり、 NaN は伝播する。0 の符号については、 -0 < +0 として扱われる。

`minimumNumber` と `maximumNumber` は、NaN を「入力の欠落」として扱い、両方 NaN の場合に限り quiet NaN を返す。0 の符号については、 -0 < +0 として扱われる。

IEEE 754-2008 の `minNum` / `maxNum` の仕様の問題点を改善したのが `minimumNumber` / `maximumNumber` と言って良さそうだ。

# プログラミング言語

## C言語

### C11

ソース：N1570 (C11の最終ドラフト) 

C言語で浮動小数点数の min / max に対応する関数は `fmin` と `fmax` だ。

NaN についての挙動は、片方が NaN の場合、 NaN ではない方を返す（7.12.12.2, 7.12.12.3）。つまり、NaN は欠落として扱われる。

ただし N1570 では signaling NaN の扱いは定義しておらず、単に「NaN」と言ったら quiet NaN のことなので（Annex F.2.1）、signaling NaN が入力に含まれていた場合は「欠落」以外の動作をするかもしれない。

0 の符号の扱い（-0 < +0 と扱うかどうか）に関しては、実装に委ねられているようだ。規格には

> Ideally, `fmax` would be sensitive to the sign of zero, for example `fmax(-0.0, +0.0)` would return `+0`; however, implementation in software might be impractical.

と書かれている（Annex F.10.9.2）。

`fmax` の実装の例として

```c
{ return (isgreaterequal(x, y) || isnan(y)) ? x : y; }
```

が挙げられている（Annex F.10.9.2）。この実装例では -0 = +0 として扱う。

処理系を実装する側としては、コンパイル先の命令セットに IEEE 754-2008 の `minNum` / `maxNum` または IEEE 754-2019 の `minimumNumber` / `maximumNumber` に相当する命令があれば、それをそのまま利用できそうである。

例：AArch64のLinuxをターゲットとするGCCでは、最適化コンパイル時に `fmin` / `fmax` が相当する命令（`FMINNM`, `FMAXNM`）にコンパイルされるようだ。

### C2x（次期標準）

C言語の次期標準 C2x では最新の IEEE 754 に対応した改定が行われる模様だ。

2020年2月時点の working draft である N2478 では、 `__STDC_IEC_60559_BFP__` が定義された環境において `fmin` / `fmax` は IEEE 754-2008 の `minNum` / `maxNum` 準拠となっている (Annex F.3)。

ただ、IEEE 754-2008 の `minNum` と `maxNum` は IEEE 754-2019 では取り除かれてしまった。C2x でも IEEE 754-2019 をサポートする動きがあるようで、 IEEE 754-2019 準拠の min / max 関数を追加するプロポーザル (N2489) がある。

具体的には、 IEEE 754-2019 の `minimum` / `maximum` に対応する関数 `fminimum` / `fmaximum` および `minimumNumber` / `maximumNumber` に対応する関数 `fminimum_num` / `fmaximum_num` を追加する。
この際、 `fmin` と `fmax` は現状のまま残る。

もちろん、これらがそのまま最終的な規格に入るとは限らないので、気になる方は今後も注視していく必要がある。

ソース：[Pre Freiburg 2020 Documents](http://www.open-std.org/jtc1/sc22/wg14/www/docs/PreFreiburg2020.htm) > N2489 2020/02/23 Thomas, C2X proposal - min-max functions

## C++

`<cmath>` の `std::fmin` / `std::fmax` はC言語と同様である。

一方、 `<algorithm>` の `std::min`, `std::max` はジェネリックな関数であり、浮動小数点数特有の規定は見当たらない。コーナーケースについては「引数が等価な場合は最初の方を返す」という記述がある程度である。

つまり、浮動小数点数についても

```cpp
T min(T x, T y) { return y < x ? y : x; }
T max(T x, T y) { return x < y ? y : x; }
```

のような定義が採用され、挙動は

* 引数の片方が NaN の場合は第1引数が返される（NaN が伝播するとは限らない）
* 入力が -0 と +0 の場合は第1引数が返される

となる。

C++ユーザーの方は、**浮動小数点数について `fmin` / `fmax` と `std::min` / `std::max` は挙動が違う**ということを頭の片隅に入れておいてほしい。

## JavaScript と WebAssembly

ソース：[ECMAScript 2019 Language Specification](http://www.ecma-international.org/ecma-262/)

`Math.min` および `Math.max` は、いずれかの引数が NaN であれば NaN を返す（NaN が伝播する）。
また、 -0 < +0 として扱われる。

よって、これらは IEEE 754-2019 の `minimum` / `maximum` に近い挙動をする。
ECMAScript の仕様では signaling NaN の s の字も出てこないのでその辺は目を瞑ろう。

WebAssembly の `fmin` と `fmax` も同様である。

ソース：[Numerics - WebAssembly 1.1](https://webassembly.github.io/spec/core/exec/numerics.html) > [fmin](https://webassembly.github.io/spec/core/exec/numerics.html#op-fmin), [fmax](https://webassembly.github.io/spec/core/exec/numerics.html#op-fmax)

## LLVM

ソース：[LLVM Language Reference Manual — LLVM 10 documentation](http://llvm.org/docs/LangRef.html) > [‘llvm.minnum.*’ Intrinsic](http://llvm.org/docs/LangRef.html#llvm-minnum-intrinsic), [‘llvm.maxnum.*’ Intrinsic](http://llvm.org/docs/LangRef.html#llvm-maxnum-intrinsic), [‘llvm.minimum.*’ Intrinsic](http://llvm.org/docs/LangRef.html#llvm-minimum-intrinsic), [‘llvm.maximum.*’ Intrinsic](http://llvm.org/docs/LangRef.html#llvm-maximum-intrinsic)

LLVM の組み込み関数にも min / max に相当するものがある。

`llvm.minnum` / `llvm.maxnum` は、片方が NaN の場合は NaN ではない方を返す。両方が NaN の場合は quiet NaN を返す。つまり、signaling NaN の扱いを除いて  IEEE 754-2008 の `minNum` / `maxNum` に準拠する。C言語の `fmin` / `fmax` と互換性がある。

`llvm.minnum` と `llvm.maxnum` はC言語の `fmin` / `fmax` と同様、 +0 と -0 が与えられた時の挙動は規定していない。

一方、`llvm.minimum` / `llvm.maximum` は IEEE 754-2019 の `minimum` / `maximum` に準拠する。つまり、 NaN は伝播し、-0 < +0 と扱われる。

# 命令セット

アーキテクチャ固有のintrinsicsやインラインアセンブリを使ってゴリゴリSIMDプログラミングをやる方や、出力の命令数が極限まで削減されるような効率的なコードを書きたい方にとっては、各ターゲットがどのような挙動の min / max 命令を用意しているか気になるところだろう。ここでは、x86系、ARM、RISC-V の3つのアーキテクチャについて調べてみた。

## x86系

ソース：Intel SDM

### x87 FPU

レガシーな x87 FPU には min / max を一発で計算できる命令はなさそうだ（合ってる？）。

### SSE2

SSE2 およびそれ以降の SIMD 拡張には min / max の命令が追加された。ただし、これらの命令は IEEE 754 で規定されている演算にはそのまま対応しない。

精度やスカラー／ベクトルの別によって何種類か命令がある。

```
MINSS -- Scalar / Single-Precision
MINSD -- Scalar / Double-Precision
MINPS -- Packed / Single-Precision
MINPD -- Packed / Double-Precision
MAXSS -- Scalar / Single-Precision
MAXSD -- Scalar / Double-Precision
MAXPS -- Packed / Single-Precision
MAXPD -- Packed / Double-Precision
```

対応するintrinsicsは `_mm_(max|min)_[sp][sd]` となる。

挙動としては、入力が両方 0 の場合は符号に関わらず2番目の引数が返される（比較の際に -0 と +0 は等価として扱われる）。この際、signaling NaN は signaling NaN のまま返される。片方あるいは両方が NaN の場合は、2番目の引数が返される。

この挙動は、C言語で

```c
double min(double x, double y) { return x < y ? x : y; }
double max(double x, double y) { return x > y ? x : y; }
```

と実装したものとほぼ同等と思われる。

NaN についての扱いが異なるため、C言語の `fmin` / `fmax` を `MINSD` / `MAXSD` にそのまま対応させることはできない。どうしても使いたい場合は自前で NaN を処理する必要がある。

一方、C言語で手動で `x < y ? x : y` と書いたコードや、C++の `std::min` / `std::max` は `MIN[SP][SD]` / `MAX[SP][SD]` の一命令にコンパイルされる可能性がある。

### AVX512

AVX512DQ の `VRANGE[SP][SD]` 命令を使うと、 0 の符号や NaN をいい感じに処理することができる。具体的には、

* 入力の少なくとも一方が signaling NaN の場合は quiet NaN を返す
* 入力の片方が quiet NaN でもう片方が通常の数の場合は、通常の数を返す（quiet NaN は入力の欠落として扱われる）
* -0 < +0 として扱われる

NaN の扱いに関しては、IEEE 754-2008 の `minNum` / `maxNum` 準拠と思って良さそうだ。

サンプルコードは次の通りである：

```c
#if !defined(__AVX512DQ__)
#error AVX512DQ not enabled
#endif

#include <x86intrin.h>

double avx512_min(double x, double y)
{
    __m128d xv = _mm_set_sd(x);
    __m128d yv = _mm_set_sd(y);
    __m128d resultv = _mm_range_sd(xv, yv, 0 /* min */ | 4 /* sign from compare result */);
    double result;
    _mm_store_sd(&result, resultv);
    return result;
}
double avx512_max(double x, double y)
{
    __m128d xv = _mm_set_sd(x);
    __m128d yv = _mm_set_sd(y);
    __m128d resultv = _mm_range_sd(xv, yv, 1 /* max */ | 4 /* sign from compare result */);
    double result;
    _mm_store_sd(&result, resultv);
    return result;
}
```

## ARM

### ARMv7 (Neon)

`VMIN` / `VMAX` は入力のいずれかが（quiet または signaling）NaN なら quiet NaN を返す。0 の符号については -0 < +0 として扱う。よって、これらは IEEE 754-2019 の `minimum` / `maximum` 準拠と言って良さそうだ。

### ARMv8 (AArch64)

AArch64 には `FMIN` / `FMINNM` / `FMAX` / `FMAXNM` などの命令が用意されている。

`FMIN` / `FMAX` は、入力のいずれかが（quiet または signaling）NaN なら quiet NaN を返す。0 の符号については -0 < +0 として扱う。よって、これらは IEEE 754-2019 の `minimum` / `maximum` 準拠と言って良さそうだ。

`FMINNM` / `FMAXNM` は、quiet NaN を「欠落」として扱う。入力に signaling NaN が含まれていた場合は quiet NaN を返す。0 の符号については -0 < +0 として扱う。よって、これらは IEEE 754-2008 の `minNum` / `maxNum` 準拠と言って良さそうだ。

## RISC-V

浮動小数点数拡張に `FMIN` および `FMAX` 命令が規定されている。これらは IEEE 754-2019 の `minimumNumber` / `maximumNumber` に準拠する。つまり、入力の片方だけが NaN であれば、NaN ではない方が返される（NaN は「欠落」として扱われる）。0 の符号に関しては -0 < +0 として扱われる。

「F」拡張のバージョン2.2で、IEEE 754-2008 の `minNum` / `maxNum` ではなく IEEE 754-2019 の `minimumNumber` / `maximumNumber` に準拠するように改定されたらしい。「D」拡張と「Q」拡張も同様である。

# まとめ

各プログラミング環境の min / max の挙動を表にまとめると、次のようになる：

| | IEEE 754 への準拠 | quiet NaN | signaling NaN | -0 < +0 | 可換 | 結合的 |
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| IEEE 754-2008 minNum / maxNum | | 欠落扱い | quiet NaN として伝播 | 実装依存 | 実装依存 | No |
| IEEE 754-2019 minimum / maximum | | 伝播 | quiet NaN として伝播 | Yes | Yes | Yes |
| IEEE 754-2019 minimumNumber / maximumNumber | | 欠落扱い | 欠落扱い | Yes | Yes | Yes |
| C11 fmin / fmax | minNum / maxNum （signaling NaN の扱いを除く） | 欠落扱い | 未定義 | 実装依存 | 実装依存 | 実装依存？ |
| C++ std::min / std::max | なし | | | No | No |
| JavaScript Math.min / Math.max | minimum / maximum （signaling NaN の扱いを除く） | 伝播 | - | Yes | Yes | Yes |
| llvm.minnum / llvm.maxnum | minNum / maxNum （signaling NaN の扱いを除く） | 欠落扱い | 欠落扱い | 規定なし | 実装依存 | 実装依存？ |
| llvm.minimum / llvm.maximum | minimum / maximum | 伝播 | 伝播 | Yes | Yes | Yes |
| SSE2 MIN[SP][SD] / MAX[SP][SD] | なし | | | No | | |
| AVX512 VRANGE[SP][SD] | minNum / maxNum | 欠落扱い | quiet NaN として伝播 | Yes | Yes | No |
| ARMv7 VMIN / VMAX | minimum / maximum | 伝播 | quiet NaN として伝播 | Yes | Yes | Yes
| AArch64 FMIN / FMAX | minimum / maximum | 伝播 | quiet NaN として伝播 | Yes | Yes | Yes |
| AArch64 FMINNM / FMAXNM | minNum / maxNum | 欠落扱い | quiet NaN として伝播 | Yes | Yes | No |
| RISC-V FMIN / FMAX | minimumNumber / maximumNumber| 欠落扱い | 欠落扱い | Yes | Yes | Yes |

（何か間違いがあったら教えてください）
