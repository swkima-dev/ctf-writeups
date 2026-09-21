---
date: '2026-09-21T21:16:54+09:00'
title: 'Cylab perplexed'
tags: [ "picoGym", "medium", "reverseEngineering"]
---

## 問題リンク

- https://learn.cylabacademy.org/library/458

## 概要

アセンブリが与えられるのみ。観察するとなにかをチェックしているので、チェックが通る文字列を探せば良いと気づく。

## 予備知識

デコンパイルできるツールが扱えれば良い。

## 解き方

問題文からは何もわからないので、fileコマンドで与えられたバイナリを見ると

```
perplexed: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, 
interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=85480b12e666f376909d57d282a1ef0f30e93db4, for GNU/Linux 3.2.0, not stripped
```

x86_64機械語で書かれたLinux向けのバイナリだと判る。
テキトーに実行すると`Wrong :(`と表示される。逆アセンブルするとcheckなる関数名が見えるので、
何かしらのチェックを通るような文字列を得ることがゴールだと推測できる。

ひとまずデコンパイルしてcheck関数の中身を覗くと、概ね次のような事をしている。
（Pythonで雑に翻訳した上で、変数名などを少しわかりやすくしたもの）

```
def check(input)
  local_58 = [0] * 0x17
  local_58[0] = -0x1f
  local_58[1] = -0x59
  local_58[2] = '\x1e'
~~
中略。local_58という文字列を格納する変数を初期化している
~~
  local_58[0x15] = 'g'
  local_58[0x16] = -0xc

  local_20 = 0
  local_1c = 0
  if len(input) != 0x1b:
      print("invalid length")
      exit(1)
  for i in range(1, 0x17):
      for j in range(8):
          if local_20 == 0:
              local_20 = 1
          a = 1 << (7 - j)
          b = 1 << (7 - local_20)
          if 0 < (ord(input[local_1c]) & b) != 0 < (ord(local_58[i]) & a):
              exit(1)
          local_20 += 1
          if local_20 == 8:
              local_20 = 0
              local_1c += 1
          if local_1c == 0x1b:
              break
  # ここまで到達したらOK
  print("OK, ans = ", "".join(input))
```

2重のforループがあり、local_58の各文字の0 ~ 7ビット目までとユーザーの入力文字列の各文字の1 ~ 7ビット目までを
ガチャガチャやっていると判る。

結局のところ、`if 0 < (ord(input[local_1c]) & b) != 0 < (ord(local_58[i]) & a)`
を満たさない文字列を求めればいいとわかり、式を読み解くと

- !=の右辺が成り立つ時、`ord(input[local_1c]) & b > 0`が成り立つ
  - => `b > 0`なので、`b`で1が立つところで`ord(input[local_1c])`も1が立つ
- !=の右辺が成り立たない時、`ord(input[local_1c]) & b > 0`も成り立たない
  - => `b > 0`なので、`b`で1が立っているところで`ord(input[local_1c])`は0が立っている

これをコードで書くと、
```
# local_58の初期化は省略
local_20 = 0
local_1c = 0
ans = [0] * 0x1b
for i in range(0x17):
    for j in range(8):
        if local_20 == 0:
            local_20 = 1
        a = 1 << (7 - j)
        b = 1 << (7 - local_20)

        if isinstance(local_58[i], int):
            right_arm = 0 < local_58[i] & a
        else:
            right_arm = 0 < ord(local_58[i]) & a
        if right_arm:
            # ord(input[i]) & b > 0が成り立つ => b>0より ord(input[i]) & bでどこかに1が立つ
            ans[local_1c] += 0xff & b
        else:
            # ord(input[i]) & b > 0 が成り立たない => ord(input[i]) & b が0
            ans[local_1c] += 0x0 & b

        local_20 += 1
        if local_20 == 8:
            local_20 = 0
            local_1c += 1
        if local_1c == 0x1b:
            break
print("".join(list(map(chr, ans))))
```

となり、これを実行すると答えである`picoCTF{0n3_bi7_4t_a_7im3}`が得られる。

## 感想

ちなみに、`if 0 < (ord(input[local_1c]) & b) != 0 < (ord(local_58[i]) & a)`
の式は

"入力文字列のある文字の`7-local_20`ビット目と`local_58`のある文字の`7-j`ビット目が同じ"

とも解釈できる。これでjをインデックスとするforループをもっと簡単に解釈すると、

"`local_58`で与えられる文字列をビットに直して7ビットずつ見ていった際に出てくるASCII文字列と入力文字列が一致するか？"

を検査しているとも言える。けれども、そんなことには解いているときに気づかなかった。

まぁ解けたので良いのである。
