---
date: '2026-09-22T05:20:40+09:00'
draft: true
title: 'Cylab JITFP'
tags: [ "picoGym", "hard", "reverseEngineering"]
---

## 問題リンク

https://learn.cylabacademy.org/library/737

## 概要

なぞのserverにsshをすることができ、そこにあるバイナリをどうにかしてフラグを獲る必要がある。

## 予備知識



## 解き方

(この問題では、静的解析を試したあたりから執筆者がwriteupを読み漁った末に回答にたどり着いている。
入門者にしては問題の難易度に対して筋が良すぎるのではと思うかも知れないが、ある程度の方針を知っての手法なのであしからず。)

まず、問題文をよく読んで見る

> If we can crack the password checker on this remote host, we will be able to infiltrate deeper into this criminal organization. The catch is it only functions properly on the host on which we found it.

日本語訳すると、

> リモート上のパスワードチェッカーを破れれば、この犯罪組織により深く潜入できる。問題なのは、これが私達が見つけたリモート上でしか適切に動作しないことだ。

となる。どうやらリモートの接続先のパスワードチェッカーを攻撃すれば良いとわかる。

実際に与えられたサーバーにsshしてみると、謎のメッセージとともにホームディレクトリに`ad7e550b`という実行ファイルがあることに気づく。

まずはfileコマンドとstringsコマンドで表層解析をする。

```
ssh_host:~$ file ./ad7e550b
./ad7e550b: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, 
interpreter /lib/ld-musl-x86_64.so.1, BuildID[sha1]=d7a3d6cbf9cf240eb59d0ebba874cd3021be5a3e, stripped

ssh_host:~$ strings ./ad7e550b
/lib/ld-musl-x86_64.so.1
fflush
puts
putchar
printf
prctl
stdout
_init
_fini
sleep
__cxa_finalize
__libc_start_main
libc.musl-x86_64.so.1
~~
(中略)
~~
Usage: %s <flag>
Incorrect
Incorrect
Correct
;*3$"
GCC: (Alpine 13.2.1_git20240309) 13.2.1 20240309
~~
(略)
```

この時点で、いくつの点に気づく

1. このバイナリがLinuxで動作するx86_64の機械語であること。
2. PIE(Position Independent Executable)であること
3. strippedされたELFファイルである
4. puts, putcharやCorrect, Incorrect, Usage: %s <flag>などから、多分実行時の引数に文字列を指定して合ってたらCorrectと表示されるタイプの問題だということ
5. Alpine Linuxでコンパイルされていること

3.について、strippedされているということは機械語のシンボル情報が削除されている。シンボル情報には関数名, 変数名やデバッグに必要な行番号などの情報が含まれていて、これが削除されているので
関数名や変数名を見ることができない。

4.について、実際に実行するとわかるがコマンドライン引数を何も指定しないと`Usage: ...`が表示され、指定すると30秒ほど待たされることがわかる。

5.について、この問題ではAlpine Linuxでコンパイルされたことがめちゃめちゃ重要というわけではない。
ただ、Alpine Linuxはmuslという標準Cライブラリを利用していて、これはUbuntuがデフォルトで使っているglibcとは異なるため、与えられたバイナリをUbuntuなどのLinuxに持ってきても
実行できない。(`no such file or directory`と言われ怒られる。リンクしたい標準Cライブラリが無いからだと思われる。)
これに関してはmuslを入れれば済む話なので、muslの入れ方を調べると良い。

これ以上はリモートホスト上ではあまりわからないので、scpで与えられたバイナリをローカルに持ってきて静的解析や（上手く行かないが）動的解析してみる。
`scp -P <ここにインスタンスのPORT番号> ctf-player@dolphin-cove.picoctf.net:/home/ctf-player/ad7e550b ./`でバイナリをローカルに落とすことができ、muslが必要なことに注意しながら
ローカルで試しに実行をしてみると

```
 ❯ ./ad7e550b hoge
================================v
zsh: segmentation fault (core dumped)  ./ad7e550b hoge
```

残念ながら動かない。問題文で「リモートホストじゃないと動かないよ」と言われているのでしょうがない。
ついでにpwndbgなどでdisas mainをしようとしても、シンボル情報が削除されているのでそもそもmain関数が見当たらず、そのままでは処理の全体像すら把握できない。
(以下のような形で怒られる。)
```
pwndbg> disas main
❌️ No symbol table is loaded.  Use the "file" command.
```

一旦ghidraでデコンパイルを試みると、素直には読めないがいくつか情報を得ることができる。
まず、ghidraが解析した関数の中に`entry`というものがあり、中身は以下のようになっている。
```
void processEntry entry(undefined8 param_1,undefined4 param_2)

{
  __libc_start_main(FUN_00101982,param_2,&stack0x00000008,_init,_fini,0);
  return;
}
```

シンボル情報が削除されたバイナリではパッと見ではmain関数が見当たらないが、[こちらの記事](https://stackoverflow.com/questions/5475790/how-to-disassemble-the-main-function-of-a-stripped-application)
によると`__libc_start_main`の第一引数にあたるアドレスがmain()関数のアドレスであるとわかる。

実際に`FUN_00101982`の中身はghidra上で以下のようになっており、main関数らしくfor文やputs, printfが見つかる。

```
undefined8 FUN_00101982(int param_1,undefined8 *param_2)

{ int iVar1;
  undefined8 uVar2;
  int local_10;
  int i;
  
  prctl(0x59616d61,0xffffffffffffffff,0,0,0);
  if (param_1 == 2) {
    for (local_10 = 0; local_10 < 0x20; local_10 = local_10 + 1) {
      putchar(0x3d);
      fflush((FILE *)0x0);
    }
    puts("v");
    for (i = 0; i < 0x21; i = i + 1) {
      sleep(1);
      iVar1 = (**(code **)(&DAT_00104120 + (long)*(int *)(&DAT_00104020 + (long)i * 4) * 8))
                        ((int)*(char *)((long)i + param_2[1]));
      if (iVar1 == 0) {
        FUN_00101932(0x21 - i);
        puts("Incorrect");
        return 1;
      }
      putchar(0x2a);
      fflush((FILE *)0x0);
    }
    sleep(1);
    if (*(char *)(param_2[1] + 0x21) == '\0') {
      puts("\nCorrect");
      uVar2 = 0;
    }
    else {
      puts("\nIncorrect");
      uVar2 = 1;
    }
  }
  else {
    printf("Usage: %s <flag>\n",*param_2);
    uVar2 = 1;
  }
  return uVar2;
}
```

どうやら、`(**(code **)(&DAT_00104120 + (long)*(int *)(&DAT_00104020 + (long)i * 4) * 8))
  ((int)*(char *)((long)i + param_2[1]));` がチェックをする大本の関数らしい。
更にいうと、iの途中で`iVar==0`が起こった際（つまり、検査の途中でIncorrectだった時）に呼ばれる`FUN_00101932`は、検査すべき残りの文字数分だけsleepをするようになっている。
そのため、timeコマンドでフラグが合っているかどうかで挙動が違う事を期待してフラグを特定しにいくのは上手く行かない。



## 感想

Difficultyの通り、本当にHardな問題だった。自分にとって初めてのDifficulty Hardの問題だったのもあり、writeupを見ながら解いても非常に苦戦した。

## 参考文献

- https://kushagr17.medium.com/picoctf-2026-jitfp-d24983ef7bd0
- https://stackoverflow.com/questions/5475790/how-to-disassemble-the-main-function-of-a-stripped-application
- https://itsmonish.pages.dev/blog/jitfp-picoctf-2026/
- https://qiita.com/hrkt/items/6a2169b021e756eb32e2
