---
title: decimal型（十進小数）に夢を見ている輩が多すぎる
tags:
  - 浮動小数点数
private: false
updated_at: '2024-06-08T21:46:46+09:00'
id: dd2ac0af80993751ee5a
organization_url_name: null
slide: false
ignorePublish: false
---

よく二進浮動小数点数の問題として、

```js
> 0.1 + 0.1 + 0.1
0.30000000000000004
> 0.1 + 0.1 + 0.1 === 0.3
false
```

だとか

```js
> 4.8 - 4.7 - 0.1
-3.608224830031759e-16
> 4.8 - 4.7 - 0.1 === 0.0
false
```

みたいなのが挙げられます。これが話題になった時にSNSで見かける言説が「十進小数 (decimal) 型ならこういう問題はない」です。

ですが、decimal型は十進小数を正確に表現できるという話でしかなく、全ての実数を正確に表現できるわけではありません。例えば、 `1.0 / 3.0 * 3.0` の計算を考えてみましょう。数学的には、これはちょうど `1.0` になるはずです。

## C\#の場合

C\#には標準の `decimal` 型があります。これで `1.0 / 3.0 * 3.0` を計算してみましょう。

```csharp
decimal a = 1.0m;
decimal b = 3.0m;
decimal c = a / b * b;
Console.WriteLine("{0} {1} {2}", a, b, c);
```

実行結果：

```
1.0 3.0 0.9999999999999999999999999999
```

`1.0` にはなりませんでした。

C\#の `decimal` は何者なのかというと、整数の仮数部 $M$ と指数部 $e$ について

$$
M\times 10^e
$$

と表現できる実数のことです。$M$ は絶対値が $2^{29}-1=79228162514264337593543950335$ 以下の整数で、$e$ は $-28$ 以上 $0$ 以下の整数です。

`decimal` が表現できる桁数は28桁程度と固定です。この点は、後述するJavaやPythonとの大きな違いでしょう。**任意精度ではない**のです。

また、通常の `double` と比べると、`decimal` 型は**表現できる値の範囲が狭い**（`double` は $10^{308}$ のような巨大な数を表現できるが、`decimal` は $10^{28}$ 程度が限界）ことに注意してください。科学計算には不向きでしょう。

参照：

* [Types - C# language specification | Microsoft Learn](https://learn.microsoft.com/ja-jp/dotnet/csharp/language-reference/language-specification/types#838-the-decimal-type)

## Javaの場合

Javaには `BigDecimal` 型があります。これで `1.0 / 3.0 * 3.0` を計算してみましょう。

```java
jshell> new BigDecimal("1.0").divide(new BigDecimal("3.0")).multiply(new BigDecimal("3.0"))
|  例外java.lang.ArithmeticException: Non-terminating decimal expansion; no exact representable decimal result.
|        at BigDecimal.divide (BigDecimal.java:1736)
|        at (#1:1)
```

エラーが出ました。不正確な結果を返すくらいならエラーにしてしまえ、ということですかね。嫌いではないです。

演算時に `MathContext` 等の引数を適宜指定すると、指定した精度で計算してくれます。

```java
jshell> new BigDecimal("1.0").divide(new BigDecimal("3.0"), MathContext.DECIMAL128).multiply(new BigDecimal("3.0"))
$2 ==> 0.99999999999999999999999999999999990
```

Javaの `BigDecimal` は何者なのかというと、**任意精度の十進小数**です。つまり、整数の仮数部 $M$ と指数部 $e$ について

$$
M\times 10^e
$$

と表現できる実数のことです。$M$ は多倍長整数で、$e$ は32ビット整数です。

参照：

* [BigDecimal (Java Platform SE 8)](https://docs.oracle.com/javase/8/docs/api/java/math/BigDecimal.html)

## Pythonの場合

Pythonにも `Decimal` クラスがあります。これで `1.0 / 3.0 * 3.0` を計算してみましょう。

```python
>>> from decimal import Decimal
>>> Decimal("1.0") / Decimal("3.0") * Decimal("3.0")
Decimal('0.9999999999999999999999999999')
```

`1.0` にはなりませんでした。

Pythonの `Decimal` も**任意精度の十進小数**です。デフォルトの精度は `getcontext()` 関数によって決まり、これの初期値は28桁です。100桁で計算する例は次のようになります：

```python
>>> from decimal import getcontext
>>> getcontext()
Context(prec=28, rounding=ROUND_HALF_EVEN, Emin=-999999, Emax=999999, capitals=1, clamp=0, flags=[], traps=[InvalidOperation, DivisionByZero, Overflow])
>>> getcontext().prec = 100
>>> Decimal("1.0") / Decimal("3.0") * Decimal("3.0")
Decimal('0.9999999999999999999999999999999999999999999999999999999999999999999999999999999999999999999999999999')
```

参照：

* [decimal — Decimal fixed point and floating point arithmetic — Python 3.12.4 documentation](https://docs.python.org/3/library/decimal.html)

## C言語の場合（IEEE 754の十進浮動小数点数）

C23にはオプショナルな機能としてIEEE 754準拠の十進浮動小数点数型のサポートが入ります。`_Decimal32` とか `_Decimal64` とか `_Decimal128` のような型名です。

執筆時点では、GCCが実験的にサポートしています。これで `1.0 / 3.0 * 3.0` を計算してみましょう。

```c
#include <stdio.h>

// C23がいい感じに実装された未来では strtod128 で文字列化できるが、現時点のlibcではサポートされていないようなので自前でやる
void print_d128(_Decimal128 x)
{
    if (x < 0.0dl || 1.0dl / x < 0.0dl) {
        putchar('-');
        x = -x;
    }
    unsigned long long int intPart = (unsigned long long int)x;
    printf("%llu.", intPart);
    x -= (_Decimal128)intPart;
    while (x != 0.0dl) {
        x *= 10.dl;
        intPart = (unsigned long long int)x;
        putchar('0' + intPart);
        x -= (_Decimal128)intPart;
    }
}

int main()
{
    _Decimal128 a = 1.0dl;
    _Decimal128 b = 3.0dl;
    _Decimal128 c = a / b * b;
    print_d128(c);
    putchar('\n');
}
```

```
$ gcc -std=c2x test.c
$ ./a.out
0.9999999999999999999999999999999999
```

`1.0` にはなりませんでした。

`_Decimal128` は何者なのかというと、整数の仮数部 $M$ と指数部 $e$ について

$$
M\times 10^{e-33}
$$

と表現できる実数のことです。ただし、$M$ は絶対値が $10^{34}-1$ 以下の整数（精度34桁）で、$e$ は $-6143$ 以上 $6144$ 以下の整数です。

`_Decimal64` の精度は16桁、`_Decimal32` の精度は7桁です。いずれにせよ**任意精度ではない**ことに注意してください。

IEEE 754準拠の十進浮動小数点数をハードウェア実装しているのは、現時点ではIBMのCPUくらいしかなさそうです（筆者が把握している限り）。**十進小数が好きすぎて仕方がない人はIBMの株とCPUを買いましょう。**

## 「浮動小数点数」だからいけないのか？固定小数点数だとどうなのか？

SNSを観察していると、「固定小数点数が最強！浮動小数点数はクソ！」とでも言いたげな意見を見かける気がします（印象です）。

では十進固定小数点数で `1.0 / 3.0 * 3.0` を計算してみましょう。Haskellの [Data.Fixed](https://hackage.haskell.org/package/base-4.20.0.1/docs/Data-Fixed.html) を使います。

```haskell
ghci> import Data.Fixed
ghci> 1.0 / 3.0 * 3.0 :: Fixed E12
0.999999999999
```

はい。1.0にはなりませんでした。**固定小数点数とはいえ有限の桁数で実数を表現することに変わりはない**ので、割り算とかで循環小数になるとアウトです。

そもそも、実数の表現で「二進」とか「十進」とか呼んでいるのは**表現の際に有限桁で打ち切るから**であり、**無限精度の実数表現であれば「二進」とか「十進」とかつける必要はない**のです。多倍長整数型や有理数型に「decimal」とついているのを見たことがありますか？**「十進小数」（decimal）と呼ばれるデータ型があったらその時点で正確さが何らかの形で犠牲となることは確定している**のです。

## 有理数などの正確な表現方法

小数ベースの表現方法ではなく、分母分子を多倍長整数で表現した**有理数型**を用意している言語はちょいちょいあります。Haskellの `Rational`、Rubyの `Rational`、Pythonの `Fraction` などです。有理数型であれば四則演算の際に誤差が出ることはありません。四則演算で済む用途で誤差が許されない場合は有理数型を使うと良いでしょう。

しかし、有理数の範囲では平方根とか指数関数とか三角関数が自由に計算できません。平方根とか、超越関数の一部を代数的に取り扱いたいとなると、数式処理システムに片足を突っ込むことになるでしょう。あるいは、**計算可能実数**を使えば表現力の問題はなくなりますが、今度は比較演算が停止するとは限らなくなります。

結局のところ、計算機上で実数を表現するための万能で使いやすい方法はないのです。用途に応じて最適な表現方法を考える必要があるのです。

実数の厄介さを何も知らんのに「二進だからだめ！小数点が動くからだめ！」と喚いている蒙昧な輩がSNSには多すぎませんか？
