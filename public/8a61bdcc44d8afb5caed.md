---
title: C言語で四倍精度浮動小数点数 (binary128) を使う
tags:
  - GCC
  - IEEE754
  - 浮動小数点数
private: false
updated_at: '2021-03-05T20:42:36+09:00'
id: 8a61bdcc44d8afb5caed
organization_url_name: null
slide: false
ignorePublish: false
---
C言語で精度113ビットの四倍精度浮動小数点型を使う話をします。

* 浮動小数点数の規格（IEEE 754）
* プログラミング言語の規格（C言語）
* 実際の言語処理系（GCC, Clang）
* アーキテクチャーごとのABIとハードウェア実装

のレイヤーからそれぞれ解説します。

# IEEE 754のbinary128

IEEE 754-2008以降でbinary128が規定されています。パラメーターは$b=2$, $p=113$, 指数部の最大値$\mathit{emax}=16383$です。

「四倍精度 (quad precision)」という用語はIEEEの規格書には出てきませんが、この記事では「四倍精度」をbinary128と同じ意味で使うことにします。

表現可能な範囲や精度に関していくつか値を挙げてみると、

* 最小の正の非正規化数は `0x1p-16494 = 6.475...e-4966`、
* 最小の正の正規化数は `0x1p-16382 = 3.3621...e-4932`、
* 有限の最大値は `0x1.ffff ffff ffff ffff ffff ffff ffffp16383 = 1.1897...e4932`
* 1の次に大きな数は `1 + 0x1p-112 = 1.0000 0000 0000 0000 0000 0000 0000 0000 0192 59...`

となります。

十進でいうと三十数桁程度の精度を持っている、ということになるでしょうか（倍精度では十進で十数桁でした）。$\log_{10}(2^{113})=34.0...$, $\log_{10}(2^{53})=15.95...$ です。

# long double

いくつかの環境では `long double` がbinary128です。具体的には、AArch64 Linux, RISC-V Linux等です。

詳しくは

* [long doubleの話](https://qiita.com/mod_poppo/items/8860505f38e2997cd021)

を参照してください。

# C言語的な話：TS 18661-3

現行のC標準（C11/C17）が参照している浮動小数点数の規格はIEEE 754-1985（相当）であり、当然binary128のサポートはありません。

ですが、2015年に出たTechnical SpecificationのTS 18661-3で、IEEE 754-2008で規定された追加の二進浮動小数点型が規定されています。

* [ISO - ISO/IEC TS 18661-3:2015 - Information Technology — Programming languages, their environments, and system software interfaces — Floating-point extensions for C — Part 3: Interchange and extended types](https://www.iso.org/standard/65615.html)
    * 筆者は出版された版を持っていないので、N1945を参照しながらこの記事を書いています。

追加の二進浮動小数点数型というのは、

```
_Float16
_Float32
_Float64
_Float128
```

というようなやつです。これらは基本的に必須ではありませんが、`__STDC_IEC_60559_BFP__` と `__STDC_IEC_60559_TYPES__` を定義する実装は `_Float32` と `_Float64` を提供しなければなりません。この他、 `_Float64x` みたいな末尾に `x` がつくやつも規定されていますが、本題から逸れるのでここでは解説しません。

この記事の主題は `_Float128` なわけですが、ちょっとだけ寄り道をしておくと精度が低い方の `_Float16` はARMのC拡張(ACLE)で言及されたりしています。

`_FloatN` 型と既存の `float`, `double`, `long double` とは、たとえ同じ浮動小数点形式であっても型としては異なります。つまり、 `float` 型がIEEE binary32である環境であっても `float` と `_Float32` は異なる型ですし、 `double` 型がIEEE binary64である環境であっても `double` と `_Float64` は異なる型です。また、`long double` がIEEE binary128であっても `long double` と `_Float128` は異なる型です。

リテラルのサフィックスは `fN` です。例えば `1.0f128` という感じです。

`_FloatN` 用の標準ライブラリーの関数は `__STDC_WANT_IEC_60559_TYPES_EXT__` を事前に定義した上で `<math.h>` 等を `#include` すると使えるようになります。数学関数の `float` 版のサフィックスが `f`、`long double` 版のサフィックスが `l` だったのと同じノリで、`_Float128` 版の関数には `f128` というサフィックスがつきます。

```c
_Float128 sqrtf128(_Float128 x);
_Float128 expf128(_Float128 x);
```

という感じです。

次期C標準であるC23にはTS 18661-3の内容はAnnexとして取り込まれる模様…ですかね？

* [N2601: C2X proposal - TS 18661-3 annex update 3](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2601.pdf)

ただ、「標準ライブラリーに短い名前の関数が増殖しすぎでは」という意見もあります。

* [N2426: Contain the floating point naming explosion](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2426.pdf)
    * 関連する議事録は[N2509](http://www.open-std.org/jtc1/sc22/wg14/www/docs/n2509.pdf)

この辺に関しては、今後の動向を見守っていく必要がありそうです。

コード例：

```c
#define __STDC_WANT_IEC_60559_TYPES_EXT__
#include <stdio.h>
#include <math.h>
#include <stdlib.h>

_Float128 f128_add(_Float128 x, _Float128 y)
{
    return x + y;
}

int main(void)
{
    _Float128 x = 1.0f128;
    _Float128 y = 0x1p-113f128;
    _Float128 sqrt2 = sqrtf128(2.0f128);
    char buf[1024];
    strfromf128(buf, sizeof(buf), "%a", f128_add(x, y));
    printf("1.0f128 + 0x1p-113f128 = %s\n", buf);
    strfromf128(buf, sizeof(buf), "%a", f128_add(x, y + y));
    printf("1.0f128 + (0x1p-113f128 + 0x1p-113f128) = %s\n", buf);
    strfromf128(buf, sizeof(buf), "%a", sqrt2);
    printf("sqrtf128(2.0f128) = %s\n", buf);
    strfromf128(buf, sizeof(buf), "%.40g", sqrt2);
    printf("sqrtf128(2.0f128) = %s\n", buf);
}
```

実行結果（例）：

```
1.0f128 + 0x1p-113f128 = 0x1p+0
1.0f128 + (0x1p-113f128 + 0x1p-113f128) = 0x1.0000000000000000000000000001p+0
sqrtf128(2.0f128) = 0x1.6a09e667f3bcc908b2fb1366ea95p+0
sqrtf128(2.0f128) = 1.414213562373095048801688724209697984347
```

# GCCのサポート

最近のGCCは四倍精度浮動小数点数をサポートしています。

ただし、一口に四倍精度と言っても「TS18661-3に則った `_Float128` 型」と「GCC独自の `__float128` 型」の2系統があり、さらに「`long double` がbinary128な環境での `long double`」も含めれば3系統があります。

`long double` がbinary128な環境では `__float128` は `long double` のエイリアス、そうでない環境では `__float128` は `_Float128` のエイリアスです。

`_Float128` の方が `__float128` よりも使える環境が幅広いです。例えばAArch64上のLinuxでは `_Float128` は使えますが `__float128` は使えません。

`__float128` のリテラルのサフィックスは `q` です。

詳しくは

* [Floating Types (Using the GNU Compiler Collection (GCC))](https://gcc.gnu.org/onlinedocs/gcc/Floating-Types.html#Floating-Types)

を参照してください。

`_Float128` や `__float128` に対する四則演算（のソフトウェアエミュレーション）はGCC付属のランタイムライブラリー (libgcc) で提供されますが、他の数学関数（`sqrt` 等）は別のライブラリーで提供されます。TS18661-3準拠の関数（`sqrtf128` 等）を提供するのはlibcの責任で、実際にglibcはそれらを実装しています。一方、GCC独自の `__float128` に対する数学関数や入出力関数はlibquadmathで提供されています：

* [GCC libquadmath](https://gcc.gnu.org/onlinedocs/libquadmath/)

libquadmathの提供する関数のサフィックスは `q` です。

libcがTS18661-3に対応していない環境（実質、Linux以外）で四倍精度浮動小数点数の数学関数を使いたければ、libquadmathを使うことになるでしょう。まあlibquadmathは `__float128` を必要とするので `__float128` が使えない環境では使えないのですが……。

# Clangのサポート

`long double` がbinary128なターゲット（`aarch64-linux-gnu`, `riscv64-linux-gnu` 等）ではきちんと `long double` が四倍精度となっています。

TS18661-3の `_Float128` には対応していません。

一部のターゲットでは `__float128` に対応しているようですが、libquadmathは付属しません。

# Intel C++ Compilerのサポート

特定のオプションを指定することで `_Quad` が使えるようになるらしい？ですが、筆者はICCを使える環境にないのでそれ以上はなんとも言えません。悪しからず。

* [How to use quadruple precision (_Quad) data type? - Intel Community](https://community.intel.com/t5/Intel-C-Compiler/How-to-use-quadruple-precision-Quad-data-type/td-p/1168090)

# アーキテクチャーごとの話

## Power ISA

Power ISA 3.0で四倍精度の演算が規定され、POWER9が実装しているようです。

ただ、GCCでコンパイルしてもデフォルトでは四倍精度の命令は使用されません。GCCでは `-mcpu=power9` という風にCPUの名前を指定するとそれっぽい命令を使うようになるようです。

## RISC-V

RISC-Vは単精度に対応するF拡張、倍精度に対応するD拡張の自然な延長で、四倍精度に対応するQ拡張を定義しています。まあオプションなので実際に搭載したハードウェアが出てくるのかって話ですが。
FPGAを使って自分で用意する方が手っ取り早いかもしれません。

ABI的な話をすると、関数呼び出しの際に四倍精度浮動小数点数の受け渡しに浮動小数点レジスターを使うかという問題があります。Q拡張を実装するハードウェアでは浮動小数点レジスターの幅が128ビットなのでレジスターで四倍精度を受け渡しできますが、D拡張までしか実装しないハードウェアでは浮動小数点レジスターの幅は64ビット止まりなので四倍精度の受け渡しができません。

ということは、RV64GQ向けのバイナリーを吐く際に関数呼び出しで四倍精度に浮動小数点レジスターを使うようなABI (LP64Q) を採用してしまうと、RV64G向けのバイナリー (LP64D) とのABI互換性が取れなくなってしまいます。

そういうわけで、

* [RISC-V ELF psABI specification](https://github.com/riscv/riscv-elf-psabi-doc/blob/master/riscv-elf.md)

ではRV64GQ向けであってもABIとしてLP64Dを採用するように「強く推奨 (strongly recommended)」しています。

なお、現在のGCC（`riscv64-linux-gnu-gcc-10 (Ubuntu 10.2.0-5ubuntu1~20.04) 10.2.0` で確認）では `-march=rv64gq` でQ拡張を有効にしても四倍精度用の命令は使用されません。

## SPARC

SPARC v8は四倍精度用の命令セットを持っているようです。ハードウェア実装されているかはともかくとして……。

https://sparc.org/technical-documents/#V8

## x86とかAArch64とか

上にあげたもの以外の今日のアーキテクチャーで四倍精度用の命令を用意しているものは、筆者の知る限りありません。ただ、ABIによっては、binary128の呼出規約（受け渡し方法）を定めているものがあります。

[x86\_64用のSystem V ABI](https://www.uclibc.org/docs/psABI-x86_64.pdf)ではbinary128の値はxmmレジスターを使って受け渡すことになっています。
一方、WindowsのABIにはそういう規定はなさそうで、他の128ビット値と同じようにポインターを経由して受け渡されるようです。

AArch64も浮動小数点レジスターが128ビットあるのでbinary128をそのまま受け渡せます。

* [Procedure Call Standard for the Arm® 64-bit Architecture (AArch64)](https://github.com/ARM-software/abi-aa/releases)

# おまけ：C++

現在のGCCの実装では、C++では `_Float128` は使えず、 `__float128` か `long double` を使うしかなさそうです。

Boost.MultiprecisionがGCCの `__float128` 型とICCの `_Quad` 型をラップしたやつを提供しているようです。

* [boost/multiprecision/float128](https://www.boost.org/doc/libs/1_75_0/libs/multiprecision/doc/html/boost_multiprecision/tut/floats/float128.html)

# 比較表

`long double` と `_Float128` と `__float128` のざっくりとした比較表を載せておきます。

|  | `long double` | `_Float128` | `__float128` |
|-:|:-:|:-:|:-:|
| 標準？ | Yes | TS18661-3 | GCC拡張 |
| 実体 | 環境依存 | binary128 | binary128 |
| リテラルのサフィックス | `l` | `f128` | `q` |
| 数学関数のサフィックス | `l` | `f128` | `q` |
| 数学関数の提供元 | libc / `<math.h>` | libc / `<math.h>` with `WANT` | libquadmath / `<quadmath.h>` |

また、筆者の独断と偏見で選んだ環境についての `long double`, `_Float128`, `__float128` の状況を挙げると

* x86\_64 Linux/glibc: `long double` は80ビット、 `_Float128`, `__float128` が利用可能。glibcは`_Float128`に対応、libquadmathも使える。
* x86\_64 mingw-w64: `long double` は80ビット、 `__float128` が利用可能。`_Float128` 型は使えるが標準ライブラリーのサポートはない。
* x86\_64 macOS: `long double` は80ビット、 `_Float128`, `__float128` が利用可能。libcは `_Float128` に非対応、数学関数にはlibquadmathも使える。
* AArch64 Linux/glibc: `long double` は四倍精度、`_Float128` が利用可能だが `__float128` は利用不可。libquadmathもない。
* PowerPC Linux/glibc: `-mlong-double-128 -mabi=ieeelongdouble` を指定すると `long double` が四倍精度になる。`_Float128` と `__float128` の両方に対応。
* RISC-V 64 Linux/glibc: `long double` は四倍精度、 `_Float128` が利用可能だが `__float128` は利用不可。libquadmathもない。

| | `long double` | `_Float128` | `__float128` |
|-:|:-:|:-:|:-:|
| x86\_64 Linux | 80ビット | OK | OK |
| x86\_64 mingw-w64 | 80ビット | libcのサポートなし | OK |
| x86\_64 macOS | 80ビット | libcのサポートなし | OK |
| AArch64 Linux | OK (binary128) | OK | 非対応 |
| PPC64 Linux | `-mabi` の指定による | OK | OK |
| RISC-V 64 Linux | OK (binary128) | OK | 非対応 |

となります。

# まとめ

GCCを使えば多くの環境で四倍精度浮動小数点数を使えます。しかし、環境によって使える書き方・ライブラリーが違うので、なるべく多くの環境で動作するような**ポータブルなコードを書くのは難しそう**です。
