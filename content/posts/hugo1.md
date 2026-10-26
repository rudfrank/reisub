---
title: 'Hands on with Hugo'
description: "Part 1."
date: '2026-10-25T13:30:51+01:00'
tags: ["hosting","hugo","git"]
author: "rudfrank"
draft: false
showToc: true
series: ["hugo-blog-deploy"]
---

## Initialize Hugo and install PaperMod theme
Install [Hugo](https://gohugo.io) (and git) on dev machine (``xbps-install -S hugo git`` on void ``apt install hugo git`` on debian, etc.) \
Hugo is a neat static site generator written in Go.

Create the site structure -- I prefer yaml configs (you can use toml): \
``hugo new site blog --format yaml``

Choose a theme. You can find one at https://themes.gohugo.io/themes \
I decided [PaperMod](https://themes.gohugo.io/themes/hugo-papermod/) will suffice for start, I like its minimalist look. We will customize it later.

Now I have to init a new git repository in the folder newly created by hugo: \
``cd blog`` then ``git init``

To [install](https://github.com/adityatelange/hugo-PaperMod/wiki/Installation#installingupdating-papermod) the selected theme, I simply add it as a submodule:
```shell
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive

```
Later it can be updated with ``git submodule update --remote --merge``

## Setting hugo.yaml
Configure the site in hugo.yaml, for template-specific parameters see the documentation https://github.com/adityatelange/hugo-PaperMod/wiki/Installation#sample-hugoyml \
Do not forget to set the theme: ``theme: ["PaperMod"]    ``

## First post
I create my 1st post (to see how it looks): ``hugo new content content/posts/hello.md`` \
it creates a markdown file, with header -- title, date and draft status. Edit it with your favorite texteditor.

The 'skeleton' file is in ``archetypes/default.md``. I clone it as post.md, then add some additional headers (like tags). Now I can create a new post based on that: ``hugo new --kind post content/posts/mynewpost.md``

## Start local development server
Run ``hugo server -D --disableFastRender``

Open the displayed url in a browser, it will auto-update on changes. Make edits, then change ``draft`` to ``false`` when ready.

