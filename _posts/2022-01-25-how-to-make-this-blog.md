---
layout: post
title:  "ブログ作った"
date:   2022-01-25 09:53:46 +0900
categories: jekyll update
---
Gihub pagesのデフォルトでjekyll使っているだけだけど、忘れるので一応メモ。
[jekyll-Quickstart]見る
```
gem install jekyll bundler
jekyll new myblog
cd myblog
bundle exec jekyll serve
```
[jekyll-gh]によると`jekyll new --skip-bundle .`でディレクトリ作った後、gem jekyllをコメントアウトして、`gem "github-pages", "~> GITHUB-PAGES-VERSION", group: :jekyll_plugins`で対応するversion入れろとのこと。

とりあえずgem "jekyll", "~> 3.9.0"にしてbundle install

使うかわからんけど`gem 'jekyll-seo-tag'`入れた。

_config.ymlにtitleとか入れた。

このページを作った。`YEAR-MONTH-DAY-title.MARKUP`の形式で_post配下に置けばOK

themaをorverrideするには_inlucde,_layout,_sassを作る。試しに_sassつくってheaderの色を変更

[jekyll-Quickstart]: https://jekyllrb.com/docs/
[jekyll-gh]:   https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll
