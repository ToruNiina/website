---
title: "Building a website powered by Hugo and LoveIt"
date:    2020-05-10T20:28:40+09:00
lastmod: 2020-05-13T19:01:40+09:00
draft: false
author: "Toru Niina"
tags: ["Hugo", "LoveIt"]
categories: ["Hugo"]
lightgallery: true
---

## Installing Hugo and LoveIt

Follow the official documentation. Note that required version of Hugo is relatively high.

- https://gohugo.io/getting-started
- https://themes.gohugo.io//theme/LoveIt/theme-documentation-basics/

## Changing Fonts

While Unicode defines CJK Unified Ideographs, actually, the forms of Japanese Kanji are slightly different from those of the Chinese characters. So we need to change fonts to render Japanese Kanji in a proper way.

We can [override the variables](https://themes.gohugo.io//theme/LoveIt/theme-documentation-basics/#34-style-customization) defined in LoveIt via `config/css/_override.scss`.
By overriding `global-font-family`, we can use fonts for Japanese letters.

```scss
$global-font-family: system-ui, -apple-system, BlinkMacSystemFont, /* fonts... */ !important;
```

Note that `!default` does not work here.
`!default` affects only when the variable is not defined or defined as null.
Since we are overriding the variable, of course it is already defined as a non-null value.

## Adding Social Icons

LoveIt supports social icons. We can find supported SNS at `[social]` table in the [site-configuration](https://themes.gohugo.io//theme/LoveIt/theme-documentation-basics/#site-configuration) section of the official document.

We can also add an icon that is not supported.
[`themes/LoveIt/assets/data/social.yml`](https://github.com/dillonzq/LoveIt/blob/master/assets/data/social.yml) defines all the supported social icons.
By adding `[params.social.XXX]` to the `config.toml` in the same structure as the yaml file, we can add social icons.

It seems that it supports [fontawesome](https://fontawesome.com/) and [simpleicons](https://simpleicons.org/).

Define `params.social.XXX.icon.class` when we use fontawesome.

```toml
[params.social.Hatena]
  weight = 4
  id = "in-neuro.hatenablog.com"
  prefix = "https://"
  title = "Hatena Blog"
  icon = {class = "fas fa-pencil-alt"}
```

Define `params.social.XXX.icon.simpleicons` when we use simpleicons.

```toml
[params.social.Qiita]
  weight = 4
  prefix = "https://qiita.com/"
  id = "niina"
  title = "Qiita"
  icon = {simpleicons = "Qiita"}
```

## Adding GitHub Star/Fork/Watch buttons

First, we can obtain buttons here -> https://buttons.github.io/.

To put raw HTML content to markdown, we need to add a shortcode.
The following post is helpful.

- https://anaulin.org/blog/hugo-raw-html-shortcode/

After adding the `rawhtml` shortcode introduced in the above post, we can put raw HTML content.

A shortcode that generates buttons may come in handy.

## Hosting on GitHub Pages

Follow the official document.

- https://gohugo.io/hosting-and-deployment/hosting-on-github/
