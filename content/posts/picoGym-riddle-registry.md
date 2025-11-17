---
date: '2025-11-17T10:42:40+09:00'
title: 'PicoGym Riddle Registry'
tags: [ "picoGym", "easy", "forensics"]
---

## 問題ページ
https://play.picoctf.org/practice/challenge/530

## 概要
捜査官としてPDFからフラグを見つけろというもの

## 解き方
PDFを開いて見える黒塗の部分はダミーなので無視
pdfをエディタで開いて、authorの部分をBase64でデコードするだけ

```
❯ echo "cGljb0NURntwdXp6bDNkX20zdGFkYXRhX2YwdW5kIV80MjQ0MGM3ZH0\075" | base64 -d
picoCTF{puzzl3d_m3tadata_f0und!_42440c7d}
```
