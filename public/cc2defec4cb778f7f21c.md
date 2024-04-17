---
title: Haskell/GHCでの浮動小数点数の扱い
tags:
  - Haskell
  - 浮動小数点数
private: false
updated_at: '2020-12-09T13:27:34+09:00'
id: cc2defec4cb778f7f21c
organization_url_name: null
slide: false
ignorePublish: false
---
Haskell/GHCでの浮動小数点数の扱いを見ていきます。

浮動小数点数特有の落とし穴や、Haskell/GHC特有の落とし穴（バグ）にも若干触れます。

# 型

Haskell標準で規定されている浮動小数点数型は

* `Float`
* `Double`
* `CFloat`
* `CDouble`

の4つです。

大抵の環境では、

* `Float` はIEEE binary32（単精度）
* `Double` はIEEE binary64（倍精度）
* `CFloat` は `Float` のnewtype
* `CDouble` は `Double` のnewtype

と思って良いでしょう。というか、GHCがそうでない環境をサポートしているか疑問です。

# リテラル

Haskellにももちろん小数リテラルがあります。

Haskellの小数リテラルはオーバーロードされており、 `Fractional a => a` という型がつきます。つまり、`0.1 :: Float` にも `0.1 :: Double` にも `0.1 :: Rational` にもなれます。

内部的には有理数 `n % m` と `fromRational` 関数を使って

```haskell
0.1 → fromRational (1 % 10)
3.14 → fromRational (157 % 50)
```

という風に脱糖されます。`fromRational` に渡される値は正確なので、型次第で好きに調理できます。

GHC 8.4では `HexFloatDecimals` 拡張として十六進小数リテラルも導入されました。また、GHC 8.6で導入された `NumericUnderscores` 拡張を使うと数値リテラルに区切り文字 `_` を入れることができます。

例：

```haskell
{-# LANGUAGE HexFloatLiterals #-}
{-# LANGUAGE NumericUnderscores #-}

main = do
  print 0x1.0000_0000_0000_1 -- 末尾の指数部分は省略可能
  print 0x1p-1024
```

実行結果：

```
1.0000000000000002
5.562684646268003e-309
```

# 型クラスと関数

各種型クラスと関数を確認します。

## Eqクラス

```haskell
instance Eq a where
  (==), (/=) :: Double -> Double -> Bool
```

等価性を比較します。浮動小数点数に関して要注意なのは

* `x == x` が成り立つとは限らない（`x` がNaNの可能性があるので）
* `x == y` だからといって `f x == f y` が成り立つとは限らない（`0` vs `-0`）

です。一方で、浮動小数点数だからといって何でも成り立たないというわけではなく、

* `x == y` と `y == x` は同値
* `not (x == y)` と `x /= y` は同値

です。

## Ordクラス

```haskell
instance Eq a => Ord a where
  compare :: Double -> Double -> Ordering
  (<), (<=), (>), (>=) :: Double -> Double -> Bool
  max, min :: Double -> Double -> Double
```

浮動小数点数には順序づけられない値、NaNがあります。なので、

* `x < y` と `not (x >= y)` は同値ではない
* `x <= y` と `not (x > y)` は同値ではない
* `x <= x` は成り立つとは限らない

です。一方で

* `x < y` と `y > x` は同値
* `x <= y` と `y >= x` は同値
* `x <= y && y <= x` と `x == y` は同値

です。

GHCでは `Float`, `Double` に対する `compare` は

```haskell
compare x y | x < y     = LT
            | x == y    = EQ
            | otherwise = GT
```

と定義されているようです。つまり、一方がNaNなら常にGTを返します。

`min`, `max` はデフォルト定義

```haskell
max x y | x <= y    = y
        | otherwise = x
min x y | x <= y    = x
        | otherwise = y
```

が採用されます。つまり、

* `(min x y, max x y)` は常に `(x, y)` または `(y, x)` のいずれかを返す
* `min 0.0 (-0.0)` は `0.0` を返し、 `max 0.0 (-0.0)` は `-0.0` を返す

となります。

## Numクラス

```haskell
instance Num a where
  (+), (-), (*) :: a -> a -> a
  negate, abs :: a -> a
  signum :: a -> a
  fromInteger :: Integer -> a
```

`(+)`, `(-)`, `(*)`, `abs` はおなじみのソレです。

`signum` は、

* 引数が正の場合は `1.0` を
* 引数が負の場合は `-1.0` を
* それ以外（±0.0, NaN）の場合は引数をそのまま

返します。

`Float`/`Double` に関する `fromInteger` は要注意で、現在のGHCでは丸め方法が一貫していません。引数が小さい場合は最近接偶数丸めを、大きい時は0方向への丸めを行います。

* [conversion from Integer to Double rounds differently depending on the value (#15926) · Issues · Glasgow Haskell Compiler / GHC · GitLab](https://gitlab.haskell.org/ghc/ghc/-/issues/15926)

実行例：

```haskell
Prelude Numeric> showHFloat (fromInteger 0xFFFFFFFFFFFFFC0 :: Double) "" -- 最近接偶数丸めが使用される
"0x1p60"
Prelude Numeric> showHFloat (fromInteger 0xFFFFFFFFFFFFFC00 :: Double) "" -- 切り捨てが行われる
"0x1.fffffffffffffp63"
```

## Fractionalクラス

```haskell
instance Num a => Fractional a where
  (/) :: a -> a -> a
  recip :: a -> a
  fromRational :: Rational -> Double
```

`(/)` はおなじみの除算です。`recip x` は `1.0 / x` として定義されています。

注意点は

* `x / y == x * recip y` は一般には成立しない
    * 例：`x = 0.01 :: Double`, `y = 0.1 :: Double` とすると `x / y == 9.999999999999999e-2`, `x * recip y == 0.1` となります。
    * どの組み合わせが反例となるかは精度に依存します。例えば、`x = 0.01 :: Float`, `y = 0.1 :: Float` の場合は `x / y == x * recip y` となります。

です。

`fromRational` は `fromInteger` と異なり、正しく最近接偶数丸めを行うようになっています。

```haskell
Prelude Numeric> showHFloat (fromRational 0xFFFFFFFFFFFFFC00 :: Double) "" -- 最近接偶数丸めが使用される
"0x1p64"
```

よって、注意点としては

* `fromRational (toRational x) == fromInteger x` は（`fromInteger` の問題により）常に成り立つとは限らない

となります。

## Floatingクラス

```haskell
instance Fractional a => Floating a where
  pi :: a
  exp, log, sqrt :: a -> a
  (**), logBase :: a -> a -> a
  sin, cos, tan :: a -> a
  asin, acos, atan :: a -> a
  sinh, cosh, tanh :: a -> a
  asinh, acosh, atanh :: a -> a

  -- GHC.Float (since base-4.9.0.0):
  log1p, expm1 :: a -> a
  log1pexp, log1mexp :: a -> a
```

おなじみの初等関数です。`sqrt` の精度は特に規定はありませんが、大抵の環境では真の値に最も近い浮動小数点数を返すと思います。

`GHC.Float` モジュールからは見慣れない関数がexportされています。

`log1p x` は $\log(1+x)$ を、 `expm1 x` は $\exp x-1$ を計算します。これらは、`x` が 0 に近いときに `log (1 + x)` や `exp x - 1` よりも正確な値を返すことが期待されます。

`log1pexp x` は $\log (1 + \exp x)$ を、 `log1mexp x` は $\log (1 - \exp x)$ を計算します。

## Realクラス

```haskell
instance (Num a, Ord a) => Real a where
  toRational :: a -> Rational
```

引数を（多倍長）有理数に変換します。整数型、有理数型、浮動小数点数型などがこのクラスのインスタンスになれます。

「有理数に正確に変換できる型」なので、実数の部分集合であっても $\mathbf{Q}(\sqrt{2})$ のように無理数を正確に表現できる型はこのクラスのインスタンスになれません。まあこの記事の本題は浮動小数点数なので、この件には深く突っ込まないでおきましょう。

浮動小数点数についての注意点としては、

* 引数が無限大やNaNの場合はめちゃくちゃな値を返す

ことです。`toRational` する際は引数が有限の浮動小数点数であることを確認しましょう。

## RealFracクラス

```haskell
instance (Real a, Fractional a) => RealFrac a where
  properFraction :: Integral b => a -> (b, a)
  truncate, round, ceiling, floor :: Integral b => a -> b
```

浮動小数点数や有理数を整数に変換します。引数が整数でなかった場合は、引数の両隣の整数のうち、

* `truncate` は絶対値が小さい方を
* `round` は最近接偶数丸めで（C言語の `round` 関数とは異なります）
* `ceiling` は大きい方を
* `floor` は小さい方を

返します。

`properFraction x` は $x=n+f$ となるような整数 $n$ と実数 $f$ を返します。$n$ は `truncate x` と等しく、$f$ の符号は $x$ と同一です。

浮動小数点数に関しての注意点は

* 引数が無限大やNaNの場合はめちゃくちゃな値が返る（`toRational` と同様）
* 変換先が固定長整数型の場合に範囲外の値を渡したら何が返ってくるかわからない

ことです。これらの関数を使う際は引数が有限の浮動小数点数であって変換先の型で表現できることを確認しましょう。

## RealFloatクラス

```haskell
instance (RealFrac a, Floating a) => RealFloat a where
  floatRadix :: a -> Integer
  floatDigits :: a -> Int
  floatRange :: a -> (Int, Int)
  decodeFloat :: a -> (Integer, Int)
  encodeFloat :: Integer -> Int -> a
  exponent :: a -> Int
  significand :: a -> a
  scaleFloat :: Int -> a -> a
  isNaN, isInfinite :: a -> Bool
  isDenormalized, isNegativeZero :: a -> Bool
  isIEEE :: a -> Bool
  atan2 :: a -> a -> a
```

浮動小数点数の中身を操作できるクラスです。

`floatRadix`, `floatDigits`, `floatRange` は浮動小数点数形式についての情報を与えます。引数型がアレですが、引数を評価しない（`undefined` を渡しても問題ない）ことを期待できます。

IEEE 754準拠の浮動小数点数型なら `floatRadix` は `2` または `10` を返すはずです。以後この値を $b$ とおきます。

`floatDigits` は仮数部の桁数 $d$ です。`Float` なら24、`Double` なら53です。ケチ表現は関係ないので注意してください。

`floatRange` は正規化数の仮数部の絶対値を $[1/b,1)$ に正規化した時の指数部の範囲です。`Float` なら `(-125,128)`, `Double` なら `(-1021,1024)` です。いずれも両端を含みます。

`decodeFloat` は浮動小数点数 $x$ を仮数部 $m$ と指数部 $e$ に分解します。$x = mb^e$ という関係が成り立ちます。$x=0$ ならば $(m,e)=(0,0)$ で、そうでなければ $b^{d-1}\le \lvert m\rvert\lt b^d$ です。

`encodeFloat` は仮数部と指数部から浮動小数点数を構築します。$m$, $e$ が与えられたときに $mb^e$ を返します。丸め方法の規定はなさそうなので、正確な値を返して欲しければ $m$ の絶対値は（正規化数の場合） $b^d$ 以下とするのが無難でしょう。

`exponent`, `significand` は仮数部の絶対値を $[1/b,1)$ に正規化した時の指数部と仮数部を返します。引数が正規化数の場合は `exponent` は `floatRange` の範囲の値を返しますが、非正規化数の場合はその範囲をはみ出ます。

`scaleFloat` は浮動小数点数に基数の冪乗を乗算します。正規化数どうしで変換する場合は正確な値を返しますが、変換後の値が非正規化数の場合（アンダーフロー）は丸めが発生します。

`isﾅﾝﾄｶ` 系のやつは特殊な浮動小数点数についての性質を確認する関数です。

`atan2 y x` は `(sin t, cos t)` の逆関数みたいなやつです。

## 冪乗

Haskellには3種類の冪乗演算子があります。

```haskell
(^) :: (Num a, Integral b) => a -> b -> a
(^^) :: (Fractional a, Integral b) => a -> b -> a
(**) :: Floating a => a -> a
```

* `(^)` は自然数乗で、任意の `Num` について適用できます。繰り返し乗算で実装されます。
* `(^^)` は整数乗で、任意の `Fractional` について適用できます。
* `(**)` は実数の実数乗あるいは、複素数の複素数乗です。

絶対値が小さい数を `(^^)` で構築する際は注意が必要です。計算途中でオーバーフロー・アンダーフローが起こる可能性があります。非正規化数の記述にはリテラルや `encodeFloat` を使うべきでしょう。

```haskell
Prelude> :set -XHexFloatLiterals
Prelude> encodeFloat 1 (-1074) :: Double
5.0e-324
Prelude> 0x1p-1074 :: Double
5.0e-324
Prelude> (1/2)^^(1074 :: Int) :: Double
5.0e-324
Prelude> 2^^(-1074 :: Int) :: Double -- !!
0.0
```

* [(^^) is not correct for Double and Float (#4413) · Issues · Glasgow Haskell Compiler / GHC · GitLab](https://gitlab.haskell.org/ghc/ghc/-/issues/4413)

## 型変換

整数と浮動小数点数の変換、浮動小数点数同士の変換のための関数がいくつか用意されています。

```haskell
-- 整数からの変換
fromInteger  :: Num b =>         Integer -> b
toInteger    :: Integral a =>          a -> Integer
fromIntegral :: (Integral a, Num b) => a -> b

-- 有理数・浮動小数点数から整数への変換
truncate :: (RealFrac a, Integral b) => a -> b
ceiling  :: (RealFrac a, Integral b) => a -> b
floor    :: (RealFrac a, Integral b) => a -> b
round    :: (RealFrac a, Integral b) => a -> b

-- 有理数を介した変換
fromRational :: Fractional b =>   Rational -> b
toRational   :: Real a =>                a -> Rational
realToFrac  :: (Real a, Fractional b) => a -> b

-- GHC.Float
float2Double :: Float -> Double
double2Float :: Double -> Float
```

`fromIntegral` は `fromInteger . toInteger` として定義されていますが、色々と書き換え規則が定義されており、`Integer` を介さない変換が可能なことも多いです。

（それでも、前述の `Float` / `Double` の `fromInteger` の問題のために書き換え規則で動作が変わるコーナーケースが存在します。）

要注意なのは `realToFrac` です。この関数は `fromRational . toRational` として定義されていますが、色々と書き換え規則が定義されており、`Rational` を介さない変換が可能な場合もあります。浮動小数点数同士の場合は、無限大やNaNを変換先の型のそれへ変換される可能性もあります。

ただ、最適化が効かない場合は `toRational` が使用されます。 `toRational` は前述の通り、入力が無限大やNaNの場合に無意味な値（何らかの有限値）を返します。それを `fromRational` に与えると、その有限値を結果の型で解釈した値を返します。

要するに、 `realToFrac` に無限大やNaNを与えるとヤバイ、ということです。次のコードを最適化ありとなしでそれぞれコンパイル・実行してみましょう：

```haskell
import Numeric

nanF :: Float
nanF = 0 / 0

nanD :: Double
nanD = 0 / 0

infinityF :: Float
infinityF = 1 / 0

main = do
  -- 最適化の有無で出力が変わる：
  putStrLn $ showHFloat (realToFrac nanF :: Double) ""
  putStrLn $ showHFloat (realToFrac infinityF :: Double) ""
  putStrLn $ showHFloat (realToFrac nanD :: Float) ""
```

最適化なし：

```shell
$ stack ghc -- RealToFrac.hs
[1 of 1] Compiling Main             ( RealToFrac.hs, RealToFrac.o )
Linking RealToFrac ...
$ ./RealToFrac
-0x1.8p128
0x1p128
-Infinity
```

最適化あり：

```shell
$ stack ghc -- -O2 RealToFrac.hs
[1 of 1] Compiling Main             ( RealToFrac.hs, RealToFrac.o ) [Optimisation flags changed]
Linking RealToFrac ...
$ ./RealToFrac
NaN
Infinity
NaN
```

最適化なしの場合は無限大やNaNが化けてしまっていることがわかります。

対策は、

* `realToFrac` の入力が常に有限値であることを確認する（無限大やNaNの場合は自前で処理する）
* `GHC.Float` モジュールの `float2Double` / `double2Float` を直接呼び出す（この場合、無限大やNaNは正しく処理される）

となるでしょう。

関連issue：

* [realToFrac doesn't sanely convert between floating types (#3676) · Issues · Glasgow Haskell Compiler / GHC · GitLab](https://gitlab.haskell.org/ghc/ghc/-/issues/3676)
* [realToFrac between Float and Double gives different results depending on compile flags (#16519) · Issues · Glasgow Haskell Compiler / GHC · GitLab](https://gitlab.haskell.org/ghc/ghc/-/issues/16519)

## 文字列との変換

`Prelude` の関数でも文字列との相互変換ができますが、フォーマット等を指定したい場合は `Numeric` モジュールの関数が使用できます。

面倒なので詳しくは触れません。

# 拙作ライブラリー

Haskell標準ではIEEE準拠の浮動小数点数を扱う道具がそれなりに揃っていますが、IEEE 754-2019で規定された演算と比較すると足りないものも色々あります。そこで、筆者が作っているのがfp-ieeeおよびrounded-hwというライブラリーです。

* [minoki/haskell-floating-point: Haskell libraries for floating point numbers](https://github.com/minoki/haskell-floating-point)
    * fp-ieee: IEEE準拠の各種演算（未リリース）
        * 丸め方法を指定できる `fromInteger` / `fromRational` 等
        * fused multiply-add
        * 四捨五入を行う `roundAway`（C言語の `round` 関数に相当）
        * `truncate`, `ceiling`, `floor`, `round`, `roundAway` の、浮動小数点数を返すバージョン
        * 「まともな」（最適化の有無で挙動が変わらない）浮動小数点数型同士の変換：`realFloatToFrac`
        * NaNのペイロードの扱い
    * rounded-hw: 丸め方法を指定した各種演算
        * 区間演算ができるようになる

fp-ieeeの方針としては、独自の型クラスはなるべく提供せずに、ジェネリックな型を持つ関数に対して書き換え規則を定義することによって個々の型に最適化された実装を用意します。また、基本的な機能はHaskellだけで完結するようにして、C FFIは最適化のためだけに利用します。

まだ足りない機能（`sqrt` の引数の型と返り値の型が異なるやつ）がありますが、Apple Silicon Mac上で動作する見込みが立ったらそろそろリリースしても良いかなと思っています。
