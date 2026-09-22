---
date: '2026-09-22T05:20:40+09:00'
title: 'Cylab JITFP'
tags: [ "picoGym", "hard", "reverseEngineering"]
---

## 問題リンク

https://learn.cylabacademy.org/library/737

## 概要

なぞのserverにsshをすることができ、そこにあるバイナリをどうにかしてフラグを獲る必要がある。

## 予備知識

強いて言うなら、バイナリ上の.bssセクションの意味や、仮想アドレスとプロセスのメモリ空間について知っていると解きやすいかもしれない。

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
4. puts, putcharやCorrect, Incorrect, Usage: %s \<flag\>などから、多分実行時の引数に文字列を指定して合ってたらCorrectと表示されるタイプの問題だということ
5. Alpine Linuxでコンパイルされていること

3.について、strippedされているということは機械語のシンボル情報が一部削除されていることを意味する。シンボル情報には関数名, 変数名などの情報が含まれていて、これが削除されているので
mainなどの名前で関数名や変数名を特定することができない。

4.について、実際に実行するとわかるがコマンドライン引数を何も指定しないと`Usage: ...`が表示され、指定すると30秒ほど待たされることがわかる。

5.について、この問題では重要ではないが、このバイナリはmuslという標準Cライブラリを利用している（`ldd`などのコマンドで確認できる）
ただ、Alpine Linuxはmuslという標準Cライブラリを利用していて、これはUbuntuがデフォルトで使っているglibcとは異なるため、与えられたバイナリをUbuntuなどのLinuxに持ってきても
実行できない。(`no such file or directory`と言われ怒られる。リンクしたい標準Cライブラリが無いからだと思われる。)

これ以上はリモートホスト上ではあまりわからないので、scpで与えられたバイナリをローカルに持ってきて静的解析や動的解析をしてみる。
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
まず、ghidraが解析した関数の中に`entry`というものがあり、その中にある処理
```
__libc_start_main(FUN_00101982,param_2,&stack0x00000008,_init,_fini,0);
```
から、メイン関数に相当する関数が`FUN_00101982`であるとわかる。このようなmain関数の特定の手法は[stackoverflowのこの質問](https://stackoverflow.com/questions/5475790/how-to-disassemble-the-main-function-of-a-stripped-application)で見つけた。
`FUN_00101982`のデコンパイルの結果を見ると以下のようになっている。
なお、変数名などは可読性のために少しだけ整えてある。
```
undefined8 FUN_00101982(int argc,undefined8 *argv)

{
  int iVar1;
  undefined8 result;
  int local_10;
  int i;
  
  prctl(0x59616d61,0xffffffffffffffff,0,0,0);
  if (argc == 2) {
    for (local_10 = 0; local_10 < 0x20; local_10 = local_10 + 1) {
      putchar(0x3d);
      fflush((FILE *)0x0);
    }
    puts("v");
    for (i = 0; i < 0x21; i = i + 1) {
      sleep(1);
      iVar1 = (**(code **)(&DAT_00104120 + (long)*(int *)(&DAT_00104020 + (long)i * 4) * 8))
                        ((int)*(char *)((long)i + argv[1]));
      if (iVar1 == 0) {
        FUN_00101932(0x21 - i);
        puts("Incorrect");
        return 1;
      }
      putchar(0x2a);
      fflush((FILE *)0x0);
    }
    sleep(1);
    if (*(char *)(argv[1] + 0x21) == '\0') {
      puts("\nCorrect");
      result = 0;
    }
    else {
      puts("\nIncorrect");
      result = 1;
    }
  }
  else {
    printf("Usage: %s <flag>\n",*argv);
    result = 1;
  }
  return result;
```

for文の中にある`(**(code **)(&DAT_00104120 + (long)*(int *)(&DAT_00104020 + (long)i * 4) * 8))
  ((int)*(char *)((long)i + argv[1]));` が、コマンドライン引数で与えられた文字列をチェックする関数らしい。
更にいうと、iの途中で`iVar==0`が起こった際（つまり、検査の途中でIncorrectだった時）に呼ばれる`FUN_00101932`は、検査すべき残りの文字数分だけsleepをするようになっている。
そのため、timeコマンドでフラグが合っているかどうかで処理時間が違う事を期待してフラグを特定しにいくのは上手く行かない。(実際に何度か実験すると、どんな文字列でも処理時間がほぼ変わらないことに気づく)

`iVar1 = (**(code **)(&DAT_00104120 + (long)*(int *)(&DAT_00104020 + (long)i * 4) * 8))
  ((int)*(char *)((long)i + argv[1]));`

は、`(int *)`がint型のポインタへの型変換であることや、`(code **)`がghidra特有の関数ポインタのポインタを表す記法だと言うことを踏まえると、以下のようなC言語とほぼ等価だとわかる

`iVar1 = function_table[index_table[i]](argv[1][i])`

ここで`&DAT_00104120`は関数ポインタが並んだ配列の先頭であり、`&DAT_00104020`はint型の数値が並んだ配列の先頭である。
つまり、パスワードをチェックする関数は関数ポインタとして`&DAT_00104120`の配列で表現されていて、アクセス先のインデックスを`&DAT_00104020`の配列の中身で1文字ずつ切り替えて
チェックしているとわかった。実際、ghidraのSymbol Tree > Functionsを見ると、以下のような1文字分のチェックを行う関数が無数にあることがわかる。

```
// こんな関数がa ~ z, A ~ Z, _, {, }の文字一つひとつについて存在
bool FUN_001011d5(char param_1)

{
  return param_1 == 'a'; 
}
```

よって、`DAT_00104120`や`DAT_00104020`がわかればチェックする関数の呼び出し順がわかり、フラグの文字列を推測できるとわかる。
そこでghidra上で探してみると

```
        DAT_00104120                                    XREF[2]:     FUN_00101982:00101a57(*), 
                                                                     FUN_00101982:00101a5e(*)  
        00104120                 ??         ??
        00104121                 ??         ??
        00104122                 ??         ??
        00104123                 ??         ??
        00104124                 ??         ??
        00104125                 ??         ??
        00104126                 ??         ??
        00104127                 ??         ??
        00104128                 ??         ??
        00104129                 ??         ??
        0010412a                 ??         ??
        0010412b                 ??         ??
```

??で埋め尽くされている！それもそのはず、バイナリの少し上の方を見ると

```

                             //
                             // .bss
                             // SHT_NOBITS  [0x40c0 - 0x4227]
                             // ram:001040c0-ram:00104227
                             //
                             DAT_001040c0                                    XREF[3]:     _FINI_0:00101140(R), 
                                                                                          _FINI_0:00101180(W), 
                                                                                          _elfSectionHeaders::00000590(*)  
        001040c0                 undefined1 ??
```

のように、アドレス`001040c0`からは.bssセクションであるとわかる。セクションとはアセンブリにおける領域の区切りの名前のことで
「.textセクションからはコード」「.dataセクションからはデータ」というように、領域名とバイナリ上のデータの内容が決まっている。
では、.bssセクションは何かというと「初期値の実データをファイルに保持せず、ロード時にゼロで初期化される静的変数を格納しておく領域」である。
.bssセクションに配置されるような変数は実行時に動的に読み書きがされるため、静的解析では値を直接知ることはできない。
この書き込み処理自体を解析できれば値を静的に求められるかも知れないが、デコンパイルした結果にはそのような処理が見当たらないため、
今回の問題ではリモート実行中の値を観測する方針をとる必要がある。

(これが本問のタイトル"JITFP"の"JIT"=Just-In-Timeの由来だと思われる)

よってまとめると、今回の問題では

- 入力文字列を1文字ずつチェックする関数が関数ポインタの配列とそのアクセス先を与える配列の2つで規定されていて、
- その関数ポインタの配列は静的解析では中身を得られず、実行時に動的に変わるので
- リモートホスト上でプログラムの実行時に動的に配列を読み取ってやる必要がある

とわかる。この時点で、残るハードルは

1. プログラムの実行時にメモリ上に配置された配列をなんとかして読み取る方法
2. exploitを実行する方法

の2つである。2点目については簡単で、sshしたリモートホスト上で`python3 --version`を打つとバージョンが返されるため、

```
ssh_host:~$ python3 -c "<ここに実行したいPythonをワンライナーで書く>"
```

のように、Pythonで書いたexploitのコードをそのままリモートホスト上で動かせると気づく。.pyのファイルをリモートホスト上で作って実行すればよいのではと思うかも知れないが、
ファイルを書き込もうとすると「Read-onlyやで」と怒られるので、上記のワンライナーでコードを実行するのが良い。
（他にも、標準入力経由でスクリプトを渡すなども可能）

1点目については、procfsの`/proc/[pid]/maps`でプロセスのメモリマップを読み取り、そこから読み取りたい仮想アドレスを割り出して`/proc/[pid]/mem`で読み取れば良い。
より具体的には、以下のように`/proc/[pid]/maps`の1行目で見れる仮想アドレスのベースアドレス(今回の場合は`6480d1197000`)を見つけ出し、

```
ssh_host:~$ ./ad7e550b hoge &> /dev/null & echo $!
[1] 16
16
ssh_host:~$ cat /proc/16/maps
6480d1197000-6480d1198000 r--p 00000000 00:32 34142925                   /home/ctf-player/ad7e550b
6480d1198000-6480d1199000 r-xp 00001000 00:32 34142925                   /home/ctf-player/ad7e550b
6480d1199000-6480d119a000 r--p 00002000 00:32 34142925                   /home/ctf-player/ad7e550b
6480d119a000-6480d119b000 r--p 00002000 00:32 34142925                   /home/ctf-player/ad7e550b
6480d119b000-6480d119c000 rw-p 00003000 00:32 34142925                   /home/ctf-player/ad7e550b
648102d5e000-648102d5f000 ---p 00000000 00:00 0                          [heap]
648102d5f000-648102d60000 rw-p 00000000 00:00 0                          [heap]
732939504000-732939508000 r--p 00000000 00:00 0                          [vvar]
732939508000-73293950a000 r--p 00000000 00:00 0                          [vvar_vclock]
73293950a000-73293950c000 r-xp 00000000 00:00 0                          [vdso]
73293950c000-732939520000 r--p 00000000 00:32 67561895                   /lib/ld-musl-x86_64.so.1
732939520000-732939574000 r-xp 00014000 00:32 67561895                   /lib/ld-musl-x86_64.so.1
732939574000-7329395aa000 r--p 00068000 00:32 67561895                   /lib/ld-musl-x86_64.so.1
7329395aa000-7329395ab000 r--p 0009d000 00:32 67561895                   /lib/ld-musl-x86_64.so.1
7329395ab000-7329395ac000 rw-p 0009e000 00:32 67561895                   /lib/ld-musl-x86_64.so.1
7329395ac000-7329395af000 rw-p 00000000 00:00 0
7fff55cc9000-7fff55cea000 rw-p 00000000 00:00 0                          [stack]
ffffffffff600000-ffffffffff601000 --xp 00000000 00:00 0                  [vsyscall]
```

そのアドレスを起点として`0x4120`分後ろにある`DAT_00104120`の配列の中身（関数ポインタは8バイトで、33文字分それがあるので264バイト分）を読めば良い。
ちなみに、`00104120`はGhidra上で今回解析した結果での仮想アドレスの場所であって、解析結果上では`00100000`からプログラムの仮想アドレスが始まるので、
考えるべきオフセットは`0x104120`ではなく`0x4120`である。
また、`DAT_00104020`の配列についても同様に読み取りができる。

あとは、得られた関数ポインタに対応するチェックされる1文字を辞書などで持って適宜参照すればよい。Pythonで実装すると、
```
import time
import subprocess
ans = ''
process = subprocess.Popen(
    ['./ad7e550b', 'A'*0x21], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
pid = process.pid
maps = open(f'/proc/{pid}/maps').read().splitlines()
base_addr = int(maps[0].split('-')[0], 16)

time.sleep(5)

fp_addr = base_addr + 0x4120
with open(f'/proc/{pid}/mem', 'rb', buffering=0) as mem:
    mem.seek(fp_addr)
    fp_data = mem.read(0x21 * 8)

index_addr = base_addr + 0x4020
with open(f'/proc/{pid}/mem', 'rb', buffering=0) as mem:
    mem.seek(index_addr)
    index_data = mem.read(0x21 * 4)

fp_offset_dict = {
    0x11d5: 'a', 0x11f2: 'b', 0x120f: 'c', 0x122c: 'd', 0x1249: 'e',
    0x1266: 'f', 0x1283: 'g', 0x12a0: 'h', 0x12bd: 'i', 0x12da: 'j',
    0x12f7: 'k', 0x1314: 'l', 0x1331: 'm', 0x134e: 'n', 0x136b: 'o',
    0x1388: 'p', 0x13a5: 'q', 0x13c2: 'r', 0x13df: 's', 0x13fc: 't',
    0x1419: 'u', 0x1436: 'v', 0x1453: 'w', 0x1470: 'x', 0x148d: 'y',
    0x14aa: 'z', 0x14c7: 'A', 0x14e4: 'B', 0x1501: 'C', 0x151e: 'D',
    0x153b: 'E', 0x1558: 'F', 0x1575: 'G', 0x1592: 'H', 0x15af: 'I',
    0x15cc: 'J', 0x15e9: 'K', 0x1606: 'L', 0x1623: 'M', 0x1640: 'N',
    0x165d: 'O', 0x167a: 'P', 0x1697: 'Q', 0x16b4: 'R', 0x16d1: 'S',
    0x16ee: 'T', 0x170b: 'U', 0x1728: 'V', 0x1745: 'W', 0x1762: 'X',
    0x177f: 'Y', 0x179c: 'Z', 0x17b9: '0', 0x17d6: '1', 0x17f3: '2',
    0x1810: '3', 0x182d: '4', 0x184a: '5', 0x1867: '6', 0x1884: '7',
    0x18a1: '8', 0x18be: '9', 0x18db: '_', 0x18f8: '{', 0x1915: '}',
}

result = []
for i in range(0x21):
    index_offset = int.from_bytes(index_data[i * 4: (i + 1) * 4], 'little')
    fp = int.from_bytes(
        fp_data[index_offset * 8: (index_offset + 1) * 8], 'little') - base_addr
    result.append(fp_offset_dict[fp])

ans = ''.join(result)
print(ans)
```

ちなみに、ssh先のターミナルでコードをそのまま貼り付けようとすると、ターミナルの文字数制限やダブルクオーテーションの扱いなどで厄介事が起こる。
なので、`base64 -w 0 solve_jitfp.py > payload_jitfp.txt`などbase64で実行したいPythonコードをエンコードしてやると、

```
ssh_host:~$ python3 -c "import base64;exec(base64.b64decode('<ここにエンコードしたPythonコードをコピペ>'))"
```

のようにエンコードした文字列を貼り付ける形でワンライナーで実行しやすくなるので良い。

さて、実際に実行してみると`1IVUCNoB4mG4kdOXZaRNVfcCX_W_Ou7w2`という結果が帰ってくる。これはフラグの形(picoCTF{hogehoge})ではないので、何かしらが間違っている。
いくらか試行錯誤すると、以下のようなことがわかる

- 何度か実行しても、5秒目で出てくる結果は毎回`1IVUCNoB4mG4kdOXZaRNVfcCX_W_Ou7w2`
- 試しに毎秒メモリをダンプして結果を見てみると、1秒ごとに結果が変わる

つまるところ、実行時に書き換わる`&DAT_00104120`から復元した結果は同じ経過時間では同一になり、
さらに関数ポインタの配列の中身は毎秒変化すると推測できる。
となると、1秒ごとに変わる配列を読みつつ、その秒数でチェックされる文字を1文字ずつ記録していけば答えにたどり着きそうである。
これをPythonのコードで実装すると以下のようになる。

```
import time
import subprocess
ans = []
process = subprocess.Popen(
    ['./ad7e550b', 'A'*0x21], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
pid = process.pid
time.sleep(0.5)
for count in range(0x21):

    maps = open(f'/proc/{pid}/maps').read().splitlines()
    base_addr = int(maps[0].split('-')[0], 16)

    fp_addr = base_addr + 0x4120
    with open(f'/proc/{pid}/mem', 'rb', buffering=0) as mem:
        mem.seek(fp_addr)
        fp_data = mem.read(0x21 * 8)

    index_addr = base_addr + 0x4020
    with open(f'/proc/{pid}/mem', 'rb', buffering=0) as mem:
        mem.seek(index_addr)
        index_data = mem.read(0x21 * 4)

    fp_offset_dict = {
        0x11d5: 'a', 0x11f2: 'b', 0x120f: 'c', 0x122c: 'd', 0x1249: 'e',
        0x1266: 'f', 0x1283: 'g', 0x12a0: 'h', 0x12bd: 'i', 0x12da: 'j',
        0x12f7: 'k', 0x1314: 'l', 0x1331: 'm', 0x134e: 'n', 0x136b: 'o',
        0x1388: 'p', 0x13a5: 'q', 0x13c2: 'r', 0x13df: 's', 0x13fc: 't',
        0x1419: 'u', 0x1436: 'v', 0x1453: 'w', 0x1470: 'x', 0x148d: 'y',
        0x14aa: 'z', 0x14c7: 'A', 0x14e4: 'B', 0x1501: 'C', 0x151e: 'D',
        0x153b: 'E', 0x1558: 'F', 0x1575: 'G', 0x1592: 'H', 0x15af: 'I',
        0x15cc: 'J', 0x15e9: 'K', 0x1606: 'L', 0x1623: 'M', 0x1640: 'N',
        0x165d: 'O', 0x167a: 'P', 0x1697: 'Q', 0x16b4: 'R', 0x16d1: 'S',
        0x16ee: 'T', 0x170b: 'U', 0x1728: 'V', 0x1745: 'W', 0x1762: 'X',
        0x177f: 'Y', 0x179c: 'Z', 0x17b9: '0', 0x17d6: '1', 0x17f3: '2',
        0x1810: '3', 0x182d: '4', 0x184a: '5', 0x1867: '6', 0x1884: '7',
        0x18a1: '8', 0x18be: '9', 0x18db: '_', 0x18f8: '{', 0x1915: '}',
    }

    result = []
    for i in range(0x21):
        index_offset = int.from_bytes(index_data[i * 4: (i + 1) * 4], 'little')
        fp = int.from_bytes(
            fp_data[index_offset * 8: (index_offset + 1) * 8], 'little') - base_addr
        result.append(fp_offset_dict[fp])

    ans.append(result[count])
    print(''.join(ans))
    time.sleep(1)
process.kill()
```

これをbase64でエンコードし、リモートホスト上で実行すると

```
ssh_host:~$ python3 -c "import base64;exec(base64.b64decode('<エンコードしたPythonコード。長過ぎるので省略>'))"
p
pi
pic
pico
picoC
picoCT
picoCTF
picoCTF{
picoCTF{p
picoCTF{pr
picoCTF{pr0
~~
中略
~~
picoCTF{pr0cf5_d36ugg3r_c303
picoCTF{pr0cf5_d36ugg3r_c3033
picoCTF{pr0cf5_d36ugg3r_c3033e
picoCTF{pr0cf5_d36ugg3r_c3033e3
picoCTF{pr0cf5_d36ugg3r_c3033e3e
picoCTF{pr0cf5_d36ugg3r_c3033e3e}
```

となり、フラグ`picoCTF{pr0cf5_d36ugg3r_c3033e3e}`が得られた。

## 感想

Difficultyの通り、本当にHardな問題だった。自分にとって初めてのDifficulty Hardの問題だったのもあり、writeupからヒントを得ながら解いても8時間程かかった。

問題を解くにあたって、私が詰まったポイント（とその解決策）は以下の通りである。

- ローカルのUbuntuではそのまま実行が出来ず、muslというCの標準ライブラリを入れる必要があること
- シンボル情報がstrippedされているためそのままではメイン関数にあたる処理がわからず、__libc_start_main()の第一引数のアドレスからメイン関数の場所を探す必要があること
- `undefined8 FUN_00101982(int param_1,undefined8 *param_2)`の`param_1`や`param_2`は、型とC言語の慣習から`argc`と`argv`だと推測できること
- C言語やghidra特有の表記。`*(int *)`や`(code **)`など
- アセンブリのセクションという概念と、.bssセクションの意味
- 実行中のプロセスのメモリの中身を覗く方法。さらに、`/proc/[pid]/mem`はプロセスの仮想アドレスでのメモリ配置を模していること
- Read-onlyなFSのみが与えられ、かつターミナルの制約やクオーテーションマークの問題を回避してワンライナーのPythonを実行する方法

問題全体を通して、C言語からコンパイルされたアセンブリの構造や、仮想アドレスを意識しつつprocfsでメモリをダンプし動的解析をする方法など、色々なことを学んだ。
記念すべき10個目のWriteupとして取り組んで良かった問題だった。

## 参考文献

- https://kushagr17.medium.com/picoctf-2026-jitfp-d24983ef7bd0
- https://itsmonish.pages.dev/blog/jitfp-picoctf-2026/
- https://stackoverflow.com/questions/5475790/how-to-disassemble-the-main-function-of-a-stripped-application
- https://stackoverflow.com/questions/1401359/understanding-linux-proc-pid-maps-or-proc-self-maps
- https://gondow.github.io/linux-x86-64-programming/3-binary.html#.text
- http://wiki.archlinux.jp/index.php/Procfs
- https://zenn.dev/juck28/articles/091c07869aba28
- https://qiita.com/hrkt/items/6a2169b021e756eb32e2
