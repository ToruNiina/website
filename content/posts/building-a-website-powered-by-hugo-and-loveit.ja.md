---
title: "Hugo + LoveItでWebサイトを作る"
date:    2020-05-10T20:28:40+09:00
lastmod: 2020-05-10T20:28:40+09:00
draft: false
author: "Toru Niina"
tags: ["Hugo", "LoveIt"]
categories: ["Hugo"]
lightgallery: true
---

とりあえずこのサイトを作った時の覚え書きを残しておきます。

## Hugo, LoveIt導入

そこそこ新しいバージョンが必要なので、brewを使わない場合はGo本体のバージョンを上げる必要があるかもしれません。

- https://gohugo.io/getting-started
- https://themes.gohugo.io//theme/LoveIt/theme-documentation-basics/

## フォント変更

フォントがデフォルトだと中華フォントになっているので、変更します。

LoveItには`config/css/_override.scss`というファイルでSCSSの[変数を上書きできる](https://themes.gohugo.io//theme/LoveIt/theme-documentation-basics/#34-style-customization)機能があります。
書き換えられるデフォルトの変数は全て`themes/LoveIt/assets/css/_variables.scss`に入っています。
ここで定義されている`global-font-family`を日本語用のものに上書きすれば解決します。

```scss
$global-font-family: system-ui, -apple-system, BlinkMacSystemFont, /*フォント...*/ !important;
```

デフォルト設定だと最後が`!default`になっていますが、`!default`だと変数が定義されていない時しか値が設定されなくなるので、すでに定義されている`$global-font-family`を上書き出来ません。

## Social icon追加

トップページにはSNSアカウントへのリンクを貼れるようになっていて、サポートされているSNSは公式ドキュメントの[site-configuration](https://themes.gohugo.io//theme/LoveIt/theme-documentation-basics/#site-configuration)の`social`に列挙されています。

デフォルトで用意されていないアイコンは設定を追加することで使えるようになります。
`themes/LoveIt/assets/data/social.yml`にサポートされている全てのSocial iconの設定方法が書いてあります。
これと同じ構造を`config.toml`の`[params.social.XXX]`に書けば新しいsocial iconを追加することができます。

見た限り、アイコンは[fontawesome](https://fontawesome.com/)と[simpleicons](https://simpleicons.org/)がサポートされており、これらにないものは`svg`ファイルを自分で用意しているようです。

fontawesomeを使う場合は`params.social.XXX.icon.class`にアイコンを指定し、

```toml
[params.social.Hatena]
  weight = 4
  id = "in-neuro.hatenablog.com"
  prefix = "https://"
  title = "Hatena Blog"
  icon = {class = "fas fa-pencil-alt"}
```

simpleiconsを使う場合は`params.social.XXX.icon.simpleicons`に名前を指定しましょう。

```toml
[params.social.Qiita]
  weight = 4
  prefix = "https://qiita.com/"
  id = "niina"
  title = "Qiita"
  icon = {simpleicons = "qiita"}
```

## GitHubのStarボタンを追加

GitHubのボタンは https://buttons.github.io/ で作れます。

普通にHTMLを書くと動きませんが、以下で紹介されているshortcodesを追加することで生HTMLを書けるようになります。

- https://anaulin.org/blog/hugo-raw-html-shortcode/

GitHub buttonを生成するshortcodesの方が便利かもしれませんが、とりあえず生HTMLで。

## GitHub Pagesにpublish

公式サイトが親切に教えてくれています。

`public/`を`git submodule`にして`user.github.io`に繋ぐというやり方ですね。

- https://gohugo.io/hosting-and-deployment/hosting-on-github/
