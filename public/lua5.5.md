---
title: Lua 5.5を触ってみる
tags:
  - Lua
private: false
updated_at: '2025-12-25T22:31:26+09:00'
id: 656690cbd5bbbf6e2d03
organization_url_name: null
slide: false
ignorePublish: false
---

この記事は「[Lua Advent Calendar 2025](https://qiita.com/advent-calendar/2025/lua)」の25日目の記事です。

2025年12月22日に、Luaの最新のメジャーバージョンである5.5.0がリリースされました。この記事では、Lua 5.5の新機能をいくつか触ってみます。詳しいことは公式マニュアル類を参照してください：

* [Lua: version history](https://www.lua.org/versions.html#5.5)
* [Lua 5.5 readme > Changes since Lua 5.4](https://www.lua.org/manual/5.5/readme.html#changes)
* [Lua 5.5 Reference Manual - contents](https://www.lua.org/manual/5.5/)
    * [Incompatibilities with the Previous Version](https://www.lua.org/manual/5.5/manual.html#8)

## `global` キーワード

従来のLuaでは、`local` で宣言した変数はレキシカルスコープを持ち、`local` で宣言されていない変数は**暗黙のグローバル変数**となっていました。

```lua
function f()
  local x = 123  -- xはlocalで宣言されているのでローカル変数
  y = x + 1  -- yは宣言されていないのでグローバル変数扱い
end
```

意図してグローバル変数を使うのであれば良いのですが、`local` 宣言を忘れた、あるいはtypoした変数が暗黙にグローバル変数になるのは、堅牢なプログラムを書く観点からは望ましくない場合もありそうです。

望ましくないグローバル変数の使用を検出する場合、従来はグローバルテーブルにメタテーブルを仕掛けて動的に検出するか、`luac` で命令列を確認して静的に検出する、などの方法がありました。

Lua 5.5には、明示的にグローバル変数を宣言する文法が用意されました。`global` 文です。`global` 文ではグローバル変数を宣言できるほか、`global` を置いたスコープでは宣言されていない変数の使用はエラーになります。

```lua
function f()
  local x = 123
  global y
  y = x + 1
end

function g()
  local x = 123
  global y = x + 1  -- 代入を伴うこともできる
end

function h(x)
  global y
  z = x + 1  -- zは宣言されていないのでエラー
end

w = 123  -- global文の影響下にないのでOK
```

変数の名前を宣言する代わりに `global *` と書くこともできて、この場合は「暗黙のグローバル」の挙動が復活します。つまり、束縛されていない変数が暗黙にグローバル変数になります。変数の属性を指定することもできて、

```lua
global<const> *
```

と書くと暗黙のグローバル変数が読み取り専用になります。`global<const> *` の影響下でも、明示的に宣言したグローバル変数は読み書きできます。

```lua
global x
global<const> *
x = 42  -- OK
```

```lua
global<const> *
global y
y = 42  -- OK
```

## 可変長引数に名前を与えられるようになった

Luaでは `...` という構文で可変長引数を扱うことができます。引数リストの最後に `...` を置くと、関数の内部で `...` という式を使うことができ、渡された可変長引数が多値として展開されます。

```lua
function printf(format, ...)
  io.write(string.format(format, ...))
end
printf("%s %g\n", "Hello Lua", 5.5)
```

可変長引数をテーブルとして利用したい場合は、`table.pack` を利用するのが通例でした。`table.pack` が登場する以前は `{n=select("#",...),...}` という式を使っていました。

```lua
function print_args(...)
  local arg = table.pack(...)
  -- arg[i]: i番目の引数
  -- arg.n: 引数の個数
  for i = 1, arg.n do
    print(arg[i])
  end
end
print_args(1, 2, 3, nil, 5)
```

今回、`...名前` という形の引数で、可変長引数をテーブルとして受け取ることができるようになりました。

```lua
function print_args(...arg)
  -- arg[i]: i番目の引数
  -- arg.n: 引数の個数
  for i = 1, arg.n do
    print(arg[i])
  end
end
print_args(1, 2, 3, nil, 5)
```

可変長引数に名前を与えた場合でも、従来通り `...` で可変長引数を展開することは可能です。

```lua
function printf(format, ...arg)
  io.write(string.format("#=%d\n", arg.n))
  io.write(string.format(format, ...))
end
printf("%s %g\n", "Hello Lua", 5.5)
-- 出力：
-- #=2
-- Hello Lua 5.5
```

## for変数が読み取り専用になった

細かい説明は不要かと思います。

```lua
for i = 1, 5 do
  i = 0  -- エラー：attempt to assign to const variable 'i'
end

for i, v in ipairs({}) do
  i = 0  -- エラー：attempt to assign to const variable 'i'
end
```

【追記】for-inの形では、最初の変数だけが読み取り専用になります。

```lua
for i,v in ipairs({}) do
  v = ""  -- エラーにならない
end
```

## 浮動小数点数の文字列化で他の数と区別できる桁数出るようになった

従来のLuaでは、浮動小数点数の `tostring` による文字列化では右の方の桁が潰れて、「値としては異なるが文字列化したら同じになる」という状況が発生していました。

```console
$ lua5.4 
Lua 5.4.8  Copyright (C) 1994-2025 Lua.org, PUC-Rio
> 1 + 1e-15
1.0
> 1 + 1e-15 == 1.0
false
```

Lua 5.5では、文字列化の際の桁数が増えて、他の浮動小数点数と区別できるような桁数が出るようになりました。

```console
$ src/lua
Lua 5.5.0  Copyright (C) 1994-2025 Lua.org, PUC-Rio
> 1 + 1e-15
1.0000000000000011
> 1 + 1e-15 == 1.0
false
> string.format("%g", 1 + 1e-15)
1
> string.format("%s", 1 + 1e-15)
1.0000000000000011
> 5e-324
4.94065645841247e-324
```

`string.format` を使う場合は、`%g` だとCライクに比較的短い桁数で文字列化されます。`%s` だと `tostring` 相当です。

実行例を見てわかる通り、一般には「文字列から数値に変換した時に同じ値になる最小の桁数」ではありません（JavaScriptやPythonと比較してみてください）。

## `table.create`: 効率的なテーブルの構築

テーブル（ハッシュテーブル）を構築する際に、事前に大きさ（要素数）がわかっていると効率的に構築できます。LuaのC APIでは、従来から `lua_createtable` 関数でテーブルの大きさ（配列パートとハッシュパート）を指定できました。

Luaで構築するテーブルでもサイズが指定できると便利な場面もありそうです。LuaJITでは `table.new` という独自拡張を実装していました。

今回、PUC Luaも `table.create` という関数でテーブルの大きさを構築時に指定できるようになりました。

```lua
function filled(n, a)
  local t = table.create(n) -- これから長さ n の配列を作る
  for i = 1, n do
    t[i] = a
  end
  return t
end
```

## `utf8.offset` 関数が終了位置を返すようになった

`utf8.offset` はUTF-8エンコードされた文字列の `i` 番目の文字（Unicodeコードポイント）の位置（バイト単位）を返す関数です。

今回、開始位置だけではなく終了位置も返すようになりました。

例えば、`"\u{3042}\u{1F600}a"` という文字列を考えると、1文字目のU+3042は1から3バイト目、2文字目のU+1F600は4バイト目から7バイト目に位置します。これと `utf8.offset` の結果を比較してみてください：

```lua
> utf8.offset("\u{3042}\u{1F600}a", 1)
1	3
> utf8.offset("\u{3042}\u{1F600}a", 2)
4	7
```

## 配列が省メモリーになった

Lua 5.3以降では、64ビット環境ではLuaの値を1個保持するのに最低16バイト使っていました。値の内容（64ビット変数とか倍精度浮動小数点数とか、ヒープに確保された領域へのポインターとか）で8バイト、値の種類で1バイトですが、配列を確保する際にはパディングという無駄な領域が7バイトあって結局16バイト使うことになります。他の処理系（LuaJITとか）だとNaN boxingみたいなトリックで8バイトに詰め込むこともできていますが、Lua 5.3以降には64ビット変数があるのでその手は使えません。

ですが、配列に関しては、値の内容とタグを別々の配列にすれば値一個につき9バイトで済みます。Lua 5.5ではこの最適化を実装して大きな配列が省メモリーになりました。

1000万要素の配列を作って、所要メモリーを比較してみましょう。計測は `time` コマンドで簡単に済ませます。macOSで試しました。

```
$ /usr/bin/time -l lua5.4 -e "local t = {}; for i = 1, 10000000 do t[i] = i end; print(#t)"
10000000
        0.41 real         0.25 user         0.15 sys
           361635840  maximum resident set size
                   0  average shared memory size
                   0  average unshared data size
                   0  average unshared stack size
               88553  page reclaims
                  48  page faults
                   0  swaps
                   0  block input operations
                   0  block output operations
                   0  messages sent
                   0  messages received
                   0  signals received
                   0  voluntary context switches
                 159  involuntary context switches
          2296811056  instructions retired
          1261024106  cycles elapsed
           352874496  peak memory footprint
```

```
$ /usr/bin/time -l src/lua -e "local t = {}; for i = 1, 10000000 do t[i] = i end; print(#t)"
10000000
        0.27 real         0.14 user         0.11 sys
           248729600  maximum resident set size
                   0  average shared memory size
                   0  average unshared data size
                   0  average unshared stack size
               61022  page reclaims
                   0  page faults
                   0  swaps
                   0  block input operations
                   0  block output operations
                   0  messages sent
                   0  messages received
                   0  signals received
                   0  voluntary context switches
                 172  involuntary context switches
          1353665704  instructions retired
           801436112  cycles elapsed
           179843072  peak memory footprint
```

peak memory footprintのところが半分近くまで減っています。良いですね。

## external string

C APIの話になりますが、Luaが管理しない外部の領域をコピーせずに文字列として使えるようになりました。`lua_pushexternalstring` 関数です。

組み込みとかのメモリーが貴重な用途や、巨大なデータを扱いたい用途で役に立つのかもしれません。

## おわり

簡単にですが、Lua 5.5の新機能を見てみました。READMEを見た感じでは他にも新機能があるようですが、各自で探索してください。

Luaはメジャーバージョンごとに破壊的変更があるので、新しいメジャーバージョンが出るということは違う亜種が増えるとも言い換えられます。移行する人は頑張って移行しましょう。LuaJITの人には関係ない話かもしれません。

【追記】ベータ版の段階で書かれたLua 5.5の新機能紹介記事があったのでリンクしておきます：[Lua 5.5(beta)の新機能紹介](https://zenn.dev/nuskey/articles/lua55-catchup)
