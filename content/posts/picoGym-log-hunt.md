---
date: '2025-11-17T12:37:10+09:00'
title: 'PicoGym Log Hunt'
tags: [ "picoGym", "easy", "generalSkills"]
---

## 問題ページ
https://play.picoctf.org/practice/challenge/527

## 問題概要
サーバーのログファイルを見て、バラバラのフラグをまとめろというもの

## 解き方
生ファイルを見ると、"INFO FLAGPART"という文字列が入ったログがフラグだと思われる

一旦grepで該当行を見る

```
❯ grep "INFO FLAGPART" server.log
[1990-08-09 10:00:10] INFO FLAGPART: picoCTF{us3_
[1990-08-09 10:02:55] INFO FLAGPART: y0urlinux_
[1990-08-09 10:05:54] INFO FLAGPART: sk1lls_
[1990-08-09 10:05:55] INFO FLAGPART: sk1lls_
[1990-08-09 10:10:54] INFO FLAGPART: cedfa5fb}
[1990-08-09 10:10:58] INFO FLAGPART: cedfa5fb}
[1990-08-09 10:11:06] INFO FLAGPART: cedfa5fb}
[1990-08-09 11:04:27] INFO FLAGPART: picoCTF{us3_
[1990-08-09 11:04:29] INFO FLAGPART: picoCTF{us3_
[1990-08-09 11:04:37] INFO FLAGPART: picoCTF{us3_
[1990-08-09 11:09:16] INFO FLAGPART: y0urlinux_
[1990-08-09 11:09:19] INFO FLAGPART: y0urlinux_
[1990-08-09 11:12:40] INFO FLAGPART: sk1lls_
[1990-08-09 11:12:45] INFO FLAGPART: sk1lls_
[1990-08-09 11:16:58] INFO FLAGPART: cedfa5fb}
[1990-08-09 11:16:59] INFO FLAGPART: cedfa5fb}
[1990-08-09 11:17:00] INFO FLAGPART: cedfa5fb}
[1990-08-09 12:19:23] INFO FLAGPART: picoCTF{us3_
[1990-08-09 12:19:29] INFO FLAGPART: picoCTF{us3_
[1990-08-09 12:19:32] INFO FLAGPART: picoCTF{us3_
[1990-08-09 12:23:43] INFO FLAGPART: y0urlinux_
[1990-08-09 12:23:45] INFO FLAGPART: y0urlinux_
[1990-08-09 12:23:53] INFO FLAGPART: y0urlinux_
[1990-08-09 12:25:32] INFO FLAGPART: sk1lls_
[1990-08-09 12:28:45] INFO FLAGPART: cedfa5fb}
[1990-08-09 12:28:49] INFO FLAGPART: cedfa5fb}
[1990-08-09 12:28:52] INFO FLAGPART: cedfa5fb}
```

重複があったりすることがわかったので、
フラグと関係ない前半37文字を削除し、sort とuniqで重複削除

先程のgrepの結果をserver-grep.logに出力させたうえで、
```
❯ cut -c 37-1000 server-grep.log | sort | uniq
 cedfa5fb}
 picoCTF{us3_
 sk1lls_
 y0urlinux_
```

なんかフラグっぽいものが出てきたので、よしなに並べ替えて
picoCTF{us3_y0urlinux_sk1lls_cedfa5fb}

