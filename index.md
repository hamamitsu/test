---
title: とりあえずJekyllで、のページ
---

# {{ page.title }}
↑Liquidを使った出力になります。
原文
````
# {{ page.title }}
````
# Liquidを使う
大文字を小文字に変換するフィルタを使ってみます。
### {{ "Hello World!" | downcase }}

````
### {{ "Hello World!" | downcase }}
````

なるほど、Liquidは本文中に埋め込んで使うものなんですね。
ちょっと勘違いしてました。
レジスタラベルに**自動的に**アドレスを付加する機能を作るのにはちょっと向かないですね。

# コンテンツを記述します


# Link
* Pages公開URL  https://hamamitsu.github.io/test/
* 練習ネタ http://jekyllrb-ja.github.io/docs/step-by-step/01-setup/
