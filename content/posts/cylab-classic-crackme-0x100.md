---
date: '2026-09-20T18:21:27+09:00'
title: 'Cylab Classic Crackme 0x100'
tags: [ "picoGym", "medium", "reverseEngineering"]
---


## 問題リンク

- https://learn.cylabacademy.org/library/409?

## 概要

よくあるcrackmeの問題で、バイナリが与えられる。

## 予備知識

ghidraなどのデコンパイラが無いと厳しいと思われるので、そういったツールの簡単な使い方を把握しておく必要がある。
ghidraについては以下のブログ記事が非常にわかりやすいので、紹介しておく。

https://daiki0508.hatenablog.com/entry/2020/06/20/185524

## 解き方

fileコマンドでどんなファイルか見てみると

```
crackme100: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, 
interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=4c56306c51af336d758655e03368b457f2f4c356, for GNU/Linux 3.2.0, with debug_info, not stripped
```

と出る。つまりx86_64の機械語なので、pwndbgなどでmain関数を逆アセンブルすると

```
...
   0x000000000040121d <+167>:   call   0x401080 <__isoc99_scanf@plt>
   0x0000000000401222 <+172>:   mov    DWORD PTR [rbp-0x4],0x0
   0x0000000000401229 <+179>:   lea    rax,[rbp-0x60]
   0x000000000040122d <+183>:   mov    rdi,rax
   0x0000000000401230 <+186>:   call   0x401040 <strlen@plt>
   0x0000000000401235 <+191>:   mov    DWORD PTR [rbp-0xc],eax
   0x0000000000401238 <+194>:   mov    DWORD PTR [rbp-0x10],0x55
   0x000000000040123f <+201>:   mov    DWORD PTR [rbp-0x14],0x33
   0x0000000000401246 <+208>:   mov    DWORD PTR [rbp-0x18],0xf
   0x000000000040124d <+215>:   mov    BYTE PTR [rbp-0x19],0x61
   0x0000000000401251 <+219>:   jmp    0x401349 <main+467>
   0x0000000000401256 <+224>:   mov    DWORD PTR [rbp-0x8],0x0
   0x000000000040125d <+231>:   jmp    0x401339 <main+451>
   0x0000000000401262 <+236>:   mov    eax,DWORD PTR [rbp-0x8]
~~
中略
~~
   0x0000000000401335 <+447>:   add    DWORD PTR [rbp-0x8],0x1
   0x0000000000401339 <+451>:   mov    eax,DWORD PTR [rbp-0x8]
   0x000000000040133c <+454>:   cmp    eax,DWORD PTR [rbp-0xc]
   0x000000000040133f <+457>:   jl     0x401262 <main+236>
   0x0000000000401345 <+463>:   add    DWORD PTR [rbp-0x4],0x1
   0x0000000000401349 <+467>:   cmp    DWORD PTR [rbp-0x4],0x2
   0x000000000040134d <+471>:   jle    0x401256 <main+224>
   0x0000000000401353 <+477>:   mov    eax,DWORD PTR [rbp-0xc]
   0x0000000000401356 <+480>:   movsxd rdx,eax
   0x0000000000401359 <+483>:   lea    rcx,[rbp-0x60]
   0x000000000040135d <+487>:   lea    rax,[rbp-0xa0]
   0x0000000000401364 <+494>:   mov    rsi,rcx
   0x0000000000401367 <+497>:   mov    rdi,rax
   0x000000000040136a <+500>:   call   0x401060 <memcmp@plt>
   0x000000000040136f <+505>:   test   eax,eax
   0x0000000000401371 <+507>:   jne    0x401389 <main+531>
   0x0000000000401373 <+509>:   mov    esi,0x402029
   0x0000000000401378 <+514>:   mov    edi,0x402040
   0x000000000040137d <+519>:   mov    eax,0x0
   0x0000000000401382 <+524>:   call   0x401050 <printf@plt>
   0x0000000000401387 <+529>:   jmp    0x401393 <main+541>
   0x0000000000401389 <+531>:   mov    edi,0x402060
   0x000000000040138e <+536>:   call   0x401030 <puts@plt>
...
```

見たところ、main+500のmemcmpで[rbp-0x60]にある文字列と何かしらの文字列の比較を行い、
それが正しければ(つまり、ZF=0であればjneでジャンプが起こらないので) main+524のprintfが
実行されてフラグを見せてくれるらしい。

実際に、pwndbgで`jump *(main+509)`などをして直接飛んで実行すると
```
pwndbg> jump *(main+509)
Continuing at 0x401373.
SUCCESS! Here is your flag: picoCTF{sample_flag}
```

と、ダミーではあるがフラグが表示された。

では、比較対象である[rbp-0xa0]を追おうとコードに再度目をやると、
込み入ったループや代入があり流石に人力で読むのは骨が折れるので、ghidraでデコンパイルする。

デコンパイルしてみると以下のようになり

```
int main(void) 
{
// このへんにはいろんな変数定義がある

  builtin_strncpy(output,"apijaczhzgtfnyjgrdvqrjbmcurcmjczsvbwgdelvxxxjkyigy",0x33);
  setvbuf(stdout,(char *)0x0,2,0);
  printf("Enter the secret password: ");
  __isoc99_scanf(&DAT_00402024,input);
  i = 0;
  sVar3 = strlen(output);
  for (; i < 3; i = i + 1) {
    for (i_1 = 0; i_1 < (int)sVar3; i_1 = i_1 + 1) {
      uVar1 = (i_1 % 0xff >> 1 & 0x55U) + (i_1 % 0xff & 0x55U);
      uVar1 = ((int)uVar1 >> 2 & 0x33U) + (uVar1 & 0x33);
      iVar2 = ((int)uVar1 >> 4) + input[i_1] + -0x61 + (uVar1 & 0xf);
      input[i_1] = (char)iVar2 + (char)(iVar2 / 0x1a) * -0x1a + 'a';
    }
  }
  iVar2 = memcmp(input,output,(long)(int)sVar3);
  if (iVar2 == 0) {
    printf("SUCCESS! Here is your flag: %s\n","picoCTF{sample_flag}");
  }
  else {
    puts("FAILED!");
  }
  return 0;
}
```

どうやら、2重のループで入力された文字列の値を何度か改変したあとに変数outputに格納
されている文字列と比較をしているらしい。

あとは2重ループの中身と逆の操作をしてmemcmp()がTrueになるような文字列を見つけ出せば良い。
多少面倒だが、正解のinputを求めるコードをPythonで書くと以下のようになる。

```
a = list("apijaczhzgtfnyjgrdvqrjbmcurcmjczsvbwgdelvxxxjkyigy")

a_len = len(a)

# 逆アセンブルした結果を雑にPythonに直すと以下のようになる。これの逆操作をすれば良い。
# for i in range(3):
#     for j in range(a_len):
#         v1 = (j % 0xff >> 1 & 0x55) + (j % 0xff & 0x55)
#         v1 = (int(v1) >> 2 & 0x33) + (v1 & 0x33)
#         v2 = (int(v1) >> 4) + ord(input[j]) + -0x61 + (v1 & 0xf)
#         input[j] = chr((v2) + int((v2 / 0x1a) * -0x1a) + ord('a'))

ans = list("apijaczhzgtfnyjgrdvqrjbmcurcmjczsvbwgdelvxxxjkyigy")
for i in reversed(range(3)):
    for j in reversed(range(a_len)):
        v1 = (j % 0xff >> 1 & 0x55) + (j % 0xff & 0x55)
        v1 = (int(v1) >> 2 & 0x33) + (v1 & 0x33)
        v2 = ord(ans[j]) - ord('a')
        ans[j] = (chr(v2 + 0x61 - (v1 & 0xf) - (v1 >> 4)))

print("".join(ans))
```
これを実行すると

amfd^]t_wan]hpa[o^phlaYa]liWd^Wkpp\na[\`poola_mZap

という結果が得られ、これをサーバーで入力してやればフラグ`picoCTF{s0lv3_angry_symb0ls_e1ad09b7}`
が得られる。

## 感想

初めてGhidraを使ったので、非常に感動。
