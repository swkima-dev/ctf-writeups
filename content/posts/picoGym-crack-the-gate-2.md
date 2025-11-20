---
date: '2025-11-20T19:05:01+09:00'
title: 'PicoGym Crack the Gate 2'
tags: [ "picoGym", "medium", "webExploitation"]
---

## 問題リンク
https://play.picoctf.org/practice/challenge/521

## 概要
ログインフォームが与えられ、「emailは特定したので
侵入してね」というもの

サイトURLとEmailとは別に、パスワードのリストも与えられます

本問は"Crack the Gate 1"という問題の続きという位置づけ

## 予備知識
### X-Forwarded-For ヘッダー

リクエストヘッダーの一種です。
端的に言えば、元のクライアントのIPアドレスをウェブサーバーに
伝えるためのHTTPヘッダーフィールドです

以下サイトがわかりやすいです
- https://developer.mozilla.org/ja/docs/Web/HTTP/Reference/Headers/X-Forwarded-For

### パスワードリスト攻撃

何らかの手法で手に入れたIDとパスワードのリストを使って
サービスに不正ログインを試みるという、攻撃手法の一種です

今回の問題では、ID固定でパスワードのリストが与えられており、
これを使って擬似的にパスワードリスト攻撃を仕掛けます

## 解き方

パスワードリストをとりあえず試すと、2回目の試行で`Too many failed attempt.
 Please try again in 20 minutes.`となってしまいブロックされます

前の問題(Crack the Gate 1)のバックドアである
`X-Dev-Acess: yes`を設定してもダメです

「システムはユーザーがコントロールするHeaderを信頼するかも」
というヒントが問題文で与えられていて、どうやら`X-Forwarded-For`の値(IPアドレス)を書き換えながら
攻撃を仕掛ければこちらのIPアドレスを偽装することができるらしいです

やり方としては、burpsuiteのIntruderタブから`Pitchfork attack`を選び、
- リクエストヘッダーの`X-Forwarded-For`を書き換え続ける
- リクエストボディの`password`フィールドでパスワードリストの全パターンを試す

ということをすればOKです

実際に攻撃を仕掛けると、以下のようなResultが得られます

|0| | |200|205|false|false|252| |
|:----|:----|:----|:----|:----|:----|:----|:----|:----|
|1|100|l9xKfsH0|200|250|false|false|252| |
|2|101|rCRnekkE|200|202|false|false|368| |
|3|102|wqMh5SQT|200|196|false|false|252| |
|4|103|9JL7BM3W|200|194|false|false|252| |
|5|104|OtrkErZU|200|209|false|false|252| |
|6|105|xr5N5yun|200|203|false|false|252| |
|7|106|FAfQ34Dr|200|261|false|false|252| |
|8|107|xAzOtoGy|200|205|false|false|252| |
|9|108|NT4Vm1FC|200|191|false|false|252| |
|10|109|aRhrp17j|200|204|false|false|252| |
|11|110|5vcxz5xZ|200|184|false|false|252| |
|12|111|SooyOtMf|200|232|false|false|252| |
|13|112|qpTlHqaG|200|207|false|false|252| |
|14|113|0AwkENeB|200|160|false|false|252| |
|15|114|tfkwkm3g|200|189|false|false|252| |
|16|115|UToyxdBs|200|204|false|false|252| |
|17|116|NWj5rDBm|200|204|false|false|252| |
|18|117|LiVR9e3g|200|306|false|false|252| |
|19|118|3v6avTIP|200|197|false|false|252| |
|20|119|jcEoe8hx|200|204|false|false|252| |

上記は攻撃1回ごとに対応するレスポンスの統計値を表しているのですが、
`Length`(つまりレスポンスの長さ)に注目すると一つだけ長さが`368`のものが見つかります

このレスポンスメッセージを見ると
```
HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: application/json; charset=utf-8
Content-Length: 132
ETag: W/"84-YU0prW3G343RIgpFiAp27XVBezo"
Date: Thu, 20 Nov 2025 09:50:30 GMT
Connection: keep-alive
Keep-Alive: timeout=5

{"success":true,
"email":"ctf-player@picoctf.org",
"firstName":"pico",
"lastName":"player",
"flag":"picoCTF{xff_byp4ss_brut3_1c447e47}"}
```

といった形でフラグが得られたのでクリアです

## 補足: 現実のサービスではこの攻撃は通用するの？

結論として、X-Forwarded-Forを書き換えてIPを偽装する攻撃は現代的な開発&デプロイを
されているサイトには通用しません
(以下、X-Forwarded-ForをXFFと略します)

理由は以下の通りです
- ブラウザは基本的にXFFを送らず、"XFFがクライアントから送られてきたらなら絶対に信用しない"がデファクトスタンダードである
- CloudflareやAWS ALBのようなサービスはクライアントが勝手に設定したXFFを破棄したり上書きする
- Epress, DjangoやRailsも*特殊な引数設定をしない限り*XFFを信用しません

よって、現代的なフレームワークを使った開発をしていて、ちゃんとしたデプロイが行われているサービスには
基本的にXFFを書き換えてIPを偽装することはできません

注意として、
- Expressでは、`app.set('trust proxy', true);`を設定するとXFFを信用してしまうらしい
- その他のフレームワークでも設定次第ではXFFを信用してしまう（かもしれない）

らしいので、バックエンド開発をする時はフレームワークによっては気をつけましょう。
特にVibe Coding時、気づかないうちに設定されていたりしたら地獄を見ます

---
## 参考文献

- https://blog.exeo-digitalsolutions.co.jp/2025/10/16/picomini-by-cmu-africa-%E5%8F%82%E5%8A%A0%E3%83%AC%E3%83%9D%E3%83%BC%E3%83%88%E5%85%A8%E5%95%8F%E8%A7%A3%E8%AA%AC/
- https://qiita.com/Taka-C/items/992ef64fd361e7ae12f9
- https://qiita.com/naka_kyon/items/8532cea02675180cb878
