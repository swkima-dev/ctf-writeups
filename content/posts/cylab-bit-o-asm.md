---
date: '2026-09-06T11:11:58+09:00'
title: 'Cylab Bit O Asm'
tags: [ "picoGym", "medium", "reverseEngineering"]
---

## 問題リンク

- https://learn.cylabacademy.org/library/391
- https://learn.cylabacademy.org/library/392
- https://learn.cylabacademy.org/library/393
- https://learn.cylabacademy.org/library/394

## 概要

x86_64のアセンブリが与えられるので、読んでeaxレジスタの値を10進数(decimal number base)で答える
というもの。
Bit-O-Asmと名前のつく問題はすべてこの形式であり、全部で4問ある。

## 予備知識

逆アセンブルの基礎についてはこちらのサイトがわかりやすい。問題の中で必要な命令(mov, add, imul, jle, jmp, ...)の解説が載っている。

- https://zenn.dev/juck28/articles/091c07869aba28

なお、今回扱う問題(Bit-O-Asm-1から4)で与えられるアセンブリは、オペランドつまりレジスタのお名前が
"eax"などになっているので、x86であるとわかる。
ちなみにARMだと

```
ADD R0, R1, R2
```
のように、英語+番号の形で記述される。

## 解き方

### Bit-O-Asm-1

与えられたファイルは以下のようになっている

```
<+0>:     endbr64
<+4>:     push   rbp
<+5>:     mov    rbp,rsp
<+8>:     mov    DWORD PTR [rbp-0x4],edi
<+11>:    mov    QWORD PTR [rbp-0x10],rsi
<+15>:    mov    eax,0x30
<+20>:    pop    rbp
<+21>:    ret
```

<+15>の行を見ると、mov命令でeaxに0x30の値を代入しているので、答えは0x30をdecimal number baseに直して `picoCTF{48}`

### Bit-O-Asm-2

与えられたファイルは以下のようになっている

```
<+0>:     endbr64
<+4>:     push   rbp
<+5>:     mov    rbp,rsp
<+8>:     mov    DWORD PTR [rbp-0x14],edi
<+11>:    mov    QWORD PTR [rbp-0x20],rsi
<+15>:    mov    DWORD PTR [rbp-0x4],0x9fe1a
<+22>:    mov    eax,DWORD PTR [rbp-0x4]
<+25>:    pop    rbp
<+26>:    ret
```

<+22>では、eaxに[rbp-0x4]の値を代入している。<+15>で[rbp-0x4]に0x9fe1aを代入していて、これはdecimal numberで654874にあたるので、
答えは`picoCTF{654874}`

捕捉として、`DWORD PTR`, `QWORD PTR`などはsize directiveと呼ばれ、アドレスの何バイトに値を移動するかを表現する。今回扱う一連の問題ではあまり重要でない。
size directive の詳細については以下が参考になる

- https://www.cs.virginia.edu/~evans/cs216/guides/x86.html

### Bit-O-Asm-3

与えられたファイルは以下のようになっている

```
<+0>:     endbr64
<+4>:     push   rbp
<+5>:     mov    rbp,rsp
<+8>:     mov    DWORD PTR [rbp-0x14],edi
<+11>:    mov    QWORD PTR [rbp-0x20],rsi
<+15>:    mov    DWORD PTR [rbp-0xc],0x9fe1a
<+22>:    mov    DWORD PTR [rbp-0x8],0x4
<+29>:    mov    eax,DWORD PTR [rbp-0xc]
<+32>:    imul   eax,DWORD PTR [rbp-0x8]
<+36>:    add    eax,0x1f5
<+41>:    mov    DWORD PTR [rbp-0x4],eax
<+44>:    mov    eax,DWORD PTR [rbp-0x4]
<+47>:    pop    rbp
<+48>:    ret
```

同様にeaxの動きを追っていけば良い。eaxがオペランドにある<+15>から<+44>までを追うと

- <+15>では[rbp-0xc]に654874を代入
- <+22>では[rbp-0x8]に4を代入
- <+29>では[rbp-0xc]の中身654874をeaxに移して
- <+32>ではeaxからimul命令でeaxに[rbp-0x8]の中身4をかけて(かけた結果は再度eaxに代入される。)
- <+36>ではeaxにさらに0x1f5つまり501を足し
- <+44>と<+47>でそれをeaxに格納している

ので、全部の操作をたどると正解は`picoCTF{2619997}`

### Bit-O-Asm-4

与えられたファイルは以下のようになっている

```
<+0>:     endbr64
<+4>:     push   rbp
<+5>:     mov    rbp,rsp
<+8>:     mov    DWORD PTR [rbp-0x14],edi
<+11>:    mov    QWORD PTR [rbp-0x20],rsi
<+15>:    mov    DWORD PTR [rbp-0x4],0x9fe1a
<+22>:    cmp    DWORD PTR [rbp-0x4],0x2710
<+29>:    jle    0x55555555514e <main+37>
<+31>:    sub    DWORD PTR [rbp-0x4],0x65
<+35>:    jmp    0x555555555152 <main+41>
<+37>:    add    DWORD PTR [rbp-0x4],0x65
<+41>:    mov    eax,DWORD PTR [rbp-0x4]
<+44>:    pop    rbp
<+45>:    ret
```

<+15>から<+29>を見ると、0x9fe1aの値と0x2710の値をcmp命令で比較して、その後にjle命令を呼び出している。
cmp命令はオペランドの左側と右側を比較してその結果をフラグという（レジスタとは別の）記憶装置で記憶していて、
今回の場合オペランドの左側が明らかに右側より大きいので、jle命令（less than equalなので左辺は右辺以下か？そうなら<main+37>へジャンプという意味）
は実行されない。

結果として、<+31>のsub命令が呼び出され、[rbp-0x4]の中身の654874から0x65すなわち101が引かれるので、答えは`picoCTF{654773}`
