---
date: '2025-11-17T16:23:57+09:00'
title: 'PicoGym Hidden in Plainsight'
tags: [ "picoGym", "easy", "forensics"]
---

## 問題リンク
https://play.picoctf.org/practice/challenge/524

## 概要
画像ファイルからフラグを探してねというもの

## 予備知識
### ステガノグラフィ
CTFにはステガノグラフィという分野があり、今回の問題はこれ

以下のサイトが役に立つはず

- https://speakerdeck.com/kuroiwasi/cpawctf-steganography?slide=63
- https://blog.hamayanhamayan.com/entry/2022/12/14/234257

## 解き方
ダウンロードしたimgファイルを普通に画像ファイルとして見ても何もわからないので、
fileコマンドを試す

```
❯ file img.jpg
img.jpg: JPEG image data, JFIF standard 1.01, aspect ratio, density 1x1, segment length 16, comment: "c3RlZ2hpZGU6Y0VGNmVuZHZjbVE9", baseline, precision 8, 640x640, components 3
```

commentが明らかに怪しいので、base64でデコードすると

```
❯ echo "c3RlZ2hpZGU6Y0VGNmVuZHZjbVE9" | base64 -d
steghide:cEF6endvcmQ=
```

steghideという文字列と、その後ろにさらに謎の文字列が出てきた

ちなみに、steghideで検索するとステガノグラフィで有名なツールの名前らしい

→「steghideでフラグが埋め込まれたjpgファイルなのでは？」と推測

"steghide:"の後ろの文字列をbase64でさらにデコードすると

```
❯ echo "cEF6endvcmQ=" | base64 -d
pAzzword
```

なんかパスワードっぽいことがわかったので、後はsteghideをインストールして
extractモードでフラグを探してみる

(steghideのインストールは省略)

使い方は簡単で、"Enter passphrase:"の後にさっきの"pAzzword"を入れるだけ
```
❯ steghide extract -sf img.jpg
Enter passphrase:
wrote extracted data to "flag.txt".
```

flag.txtの中身がpicoCTF{h1dd3n_1n_1m4g3_67479645}となっているので、多分これがフラグ
