---
date: '2026-09-09T17:41:52+09:00'
title: 'Cylab GDB baby step'
tags: [ "picoGym", "medium", "reverseEngineering"]
---

## 問題リンク

- https://learn.cylabacademy.org/library/395
- https://learn.cylabacademy.org/library/396
- https://learn.cylabacademy.org/library/397
- https://learn.cylabacademy.org/library/398

## 概要

バイナリが与えられるので、GDBを使って逆アセンブルして解読しろというもの。
GDB Baby Stepと名のつく問題はすべてこの形式であり、全部で4問ある。

ちなみに、作問者はBit-O-Asm01 ~ 04の問題と同じ LT 'syreal' Jones さんである。

## 予備知識

逆アセンブルの基礎についてはこちらのサイトがわかりやすい。pwndbgの使い方にも触れられている。

- https://zenn.dev/juck28/articles/091c07869aba28

## 解き方

### GDB baby step 1

main関数を見ると
```
   0x0000000000001129 <+0>:     endbr64
   0x000000000000112d <+4>:     push   rbp
   0x000000000000112e <+5>:     mov    rbp,rsp
   0x0000000000001131 <+8>:     mov    DWORD PTR [rbp-0x4],edi
   0x0000000000001134 <+11>:    mov    QWORD PTR [rbp-0x10],rsi
   0x0000000000001138 <+15>:    mov    eax,0x86342
   0x000000000000113d <+20>:    pop    rbp
   0x000000000000113e <+21>:    ret
```
となっているので、0x86342を10進数に直して、答えは`picoCTF{549698}`

### GDB baby step 2

main関数は以下のようになっており
```
   0x0000000000401106 <+0>:     endbr64
   0x000000000040110a <+4>:     push   rbp
   0x000000000040110b <+5>:     mov    rbp,rsp
   0x000000000040110e <+8>:     mov    DWORD PTR [rbp-0x14],edi
   0x0000000000401111 <+11>:    mov    QWORD PTR [rbp-0x20],rsi
   0x0000000000401115 <+15>:    mov    DWORD PTR [rbp-0x4],0x1e0da
   0x000000000040111c <+22>:    mov    DWORD PTR [rbp-0xc],0x25f
   0x0000000000401123 <+29>:    mov    DWORD PTR [rbp-0x8],0x0
   0x000000000040112a <+36>:    jmp    0x401136 <main+48>
   0x000000000040112c <+38>:    mov    eax,DWORD PTR [rbp-0x8]
   0x000000000040112f <+41>:    add    DWORD PTR [rbp-0x4],eax
   0x0000000000401132 <+44>:    add    DWORD PTR [rbp-0x8],0x1
   0x0000000000401136 <+48>:    mov    eax,DWORD PTR [rbp-0x8]
   0x0000000000401139 <+51>:    cmp    eax,DWORD PTR [rbp-0xc]
   0x000000000040113c <+54>:    jl     0x40112c <main+38>
   0x000000000040113e <+56>:    mov    eax,DWORD PTR [rbp-0x4]
   0x0000000000401141 <+59>:    pop    rbp
   0x0000000000401142 <+60>:    ret
```

[rbp-0x8]をインクリメントしながらcmp命令で0x25fになるまでループしているとわかる。

pythonで適当に再現すると
```
ans = 0x1e0da
limit = 0x25f
i = 0

while i < limit:
    ans += i
    i += 1

print(ans)
```
となり、実行すると307019になるので答えは`picoCTF{307019}`

### GDB baby step 3

まず問題文の英語が分かりづらいが、
byte-wiseは「1バイト毎に」という意味なので、「定数`0x2262c96b`が読み込まれたメモリは若い順から1バイト毎にどうなっていますか」
と聞いている。

x86_64がリトルエンディアンだと知っていれば、0x2262c96bを1バイトずつ(我々人間が読む順と)逆に並べて

6b c9 62 22 => `picoCTF{0x6bc96222}`

が答えだとわかる。

ちなみに、一応pwndbgを使って問題文の誘導通りに答えを得ることも可能であり、

`b main`でmain関数にブレークポイントを貼って`r`でmain関数の処理開始まで実行をし、`ni`で[rbp-0x4]に値が代入されるまで実行を進めて`x/4xb $rbp-4`でメモリの中身を表示すると

```
───────────────────────────────────────────────────────────────────[ DISASM / x86-64 / set emulate on ]────────────────────────────────────────────────────────────────────
b+ 0x40110e       <main+8>                        mov    dword ptr [rbp - 0x14], edi         [0x7fffffffda5c] <= 1
   0x401111       <main+11>                       mov    qword ptr [rbp - 0x20], rsi         [0x7fffffffda50] <= 0x7fffffffdb88 —▸ 0x7fffffffdf36 ◂— '/home/swkima/Develop/CTF/CyLab/debugger0_c'
   0x401115       <main+15>                       mov    dword ptr [rbp - 4], 0x2262c96b     [0x7fffffffda6c] <= 0x2262c96b
   0x40111c       <main+22>                       mov    eax, dword ptr [rbp - 4]            EAX, [0x7fffffffda6c] => 0x2262c96b
 ► 0x40111f       <main+25>                       pop    rbp                                 RBP => 1
   0x401120       <main+26>                       ret                                <__libc_start_call_main+128>
    ↓
   0x7ffff7c29d90 <__libc_start_call_main+128>    mov    edi, eax                            EDI => 0x2262c96b
   0x7ffff7c29d92 <__libc_start_call_main+130>    call   exit                        <exit>

   0x7ffff7c29d97 <__libc_start_call_main+135>    call   __nptl_deallocate_tsd       <__nptl_deallocate_tsd>

   0x7ffff7c29d9c <__libc_start_call_main+140>    lock dec dword ptr [rip + 0x1f0505]
   0x7ffff7c29da3 <__libc_start_call_main+147>    sete   al
~
(中略)
~
pwndbg> x/4xb $rbp-4
0x7fffffffda6c: 0x6b    0xc9    0x62    0x22
```

のように、リトルエンディアンに従って並べられた1バイト毎の値を見ることができる。

ところで、なぜ人間にとって分かりづらいリトルエンディアンが存在するかのだろうか？

その理由の一つに、低レベルなプログラミングにおける型変換が簡単になるというのがある。たとえば0x4aを32bitのメモリにリトルエンディアンで保持すると

`4a` `00` `00` `00`

と表現される。（左からアドレスが若い順）

ここで、アドレスへのアクセスは先頭アドレスでアクセスすることを鑑みると、8ビット, 16ビット, 32ビットという異なるサイズの型であっても同じアドレスでアクセスできる。
（それぞれ`0x4a`, `0x004a`, `0x0000004a`となりすべて値としては4aになる。）
これがビックエンディアンだと、同じ値にアクセスする場合でも型毎にアドレスが変わってしまうので、嬉しくない。

他にも、（ビックエンディアンと比べて）リトルエンディアンは計算効率が良いという理由もある。
これらについては[EndiannessのWikipediaのConsiderationsの節](https://en.wikipedia.org/wiki/Endianness#Considerations)が非常にわかりやすかったので、参考にされたし。

### GDB baby step 4

mainを見るとfunc1を呼び出しているので、`disas func1`をすると

```
   0x0000000000401106 <+0>:     endbr64
   0x000000000040110a <+4>:     push   rbp
   0x000000000040110b <+5>:     mov    rbp,rsp
   0x000000000040110e <+8>:     mov    DWORD PTR [rbp-0x4],edi
   0x0000000000401111 <+11>:    mov    eax,DWORD PTR [rbp-0x4]
   0x0000000000401114 <+14>:    imul   eax,eax,0x3269
   0x000000000040111a <+20>:    pop    rbp
   0x000000000040111b <+21>:    ret
```

となっている

答えは0x3269を10進数に直して、12905つまり`picoCTF{12905}`
