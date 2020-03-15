---
title: Customize Hexo and Theme NexT
date: 2020-03-15 00:15:43
categories:
- [Web, Frontend]
- Blog
tags:
- Hexo
- Hexo Theme
---

# Introduction  

我大概是从两天前开始决定使用 Hexo 的。

```
橙橙橙 2020-03-10 15:35:42
    你现在在用hexo吗
锅 2020-03-10 16:15:22
    对
```

```
橙橙橙 2020-03-12 18:05:24
    你觉得hexo好还是jekyll好
mxd 2020-03-12 18:12:17
    我喜欢hexo
```

昨天我装上了 `Hexo` ，配置了 `gh-pages` ，今天我装了一堆插件用于优化浏览体验，并把主题改成了自己喜欢的样子。我还增加了 `gitalk` 评论系统，并接入了 `Google Analytics` 服务等传统艺能。

我会在这个帖子里做一些记录，关于我修改的比较重要的部分。


# 修改 Hexo 字体

我修改了 `Hexo` 全局的字体，并根据自己的审美修改了字体大小和排版。修改字体需要重写 CSS ，覆盖 `NexT` 原有的样式表。

在 `NexT` 的配置文件中找到如下片段

```yaml
# Define custom file paths.
# Create your custom files in site directory `source/_data` and uncomment needed files below.
custom_file_path:
  #head: source/_data/head.swig
  #header: source/_data/header.swig
  #sidebar: source/_data/sidebar.swig
  #postMeta: source/_data/post-meta.swig
  #postBodyEnd: source/_data/post-body-end.swig
  #footer: source/_data/footer.swig
  #bodyEnd: source/_data/body-end.swig
  #variable: source/_data/variables.styl
  #mixin: source/_data/mixins.styl
  style: source/_data/styles.styl
```

由于我只自定义了样式表，所以我只是把样式表的部分取消注释。我希望修改我的网站全局的字体，所以我先通过 https://www.font-converter.net/ 将 ttf 字体转换成了多种格式，将其放在 `source/fonts` 文件夹下，并增加 CSS 配置。

```css
@font-face {
  font-family: "FZS3JW";
  src: url("/fonts/FZS3JW/FZS3JW.eot"); /* IE9 Compat Modes */
  src: url("/fonts/FZS3JW/FZS3JW.eot?#iefix") format("embedded-opentype"), /* IE6-IE8 */
    url("/fonts/FZS3JW/FZS3JW.otf") format("opentype"), /* Open Type Font */
    url("/fonts/FZS3JW/FZS3JW.svg") format("svg"), /* Legacy iOS */
    url("/fonts/FZS3JW/FZS3JW.ttf") format("truetype"), /* Safari, Android, iOS */
    url("/fonts/FZS3JW/FZS3JW.woff") format("woff"), /* Modern Browsers */
    url("/fonts/FZS3JW/FZS3JW.woff2") format("woff2"); /* Modern Browsers */
  font-weight: normal;
  font-style: normal;
  font-display: swap;
}

body {
    font-family: "Times New Roman", "FZS3JW", "PingFang SC", "Microsoft YaHei";
}
```

这样，全局的字体就被修改为上述优先级了。

# 评论

`NexT` 提供了非常方便的评论接口，只需要申请对应的评论服务并把他们的 `API Key` 或者别的相关的东西复制过来就可以了。我这里主要记录一下 `gitalk` 的安装过程。

## Gitalk

`gitalk` 是一个使用 `GitHub issue` 作为对话记录工具的评论，我觉得想到这个点子的人一定是天才。

首先我们需要创建一个 repo ，用于放所有的 issue （评论）。我创建了一个名为 `gitalk-wasteland` 的 repo 。然后我们什么都不用操作，进行下一步。

进入 `GitHub OAuth App` 申请页面 https://github.com/settings/applications/new ，创建一个新的 `OAuth App` 并设置 *Homepage URL* 和 *Authorization callback URL* 为 **网站首页 URL** 。然后根据配置文件注释，将 *Client ID* ， *Client Secret* ， *gitalk-wasteland(repo name)* 等信息填入

<!--more-->

```yaml
gitalk:
  enable: true
  ...
```

即可生效。这个评论模块无法在本地测试，需要部署到服务器后在之前填入的 URL 下进行测试。

不要忘记打开

```yaml
# Multiple Comment System Support
comments:
  # Available values: tabs | buttons
  style: tabs
  # Choose a comment system to be displayed by default.
  # Available values: changyan | disqus | disqusjs | gitalk | livere | valine
  active: gitalk
  ...
```

# 插件

## 搜索按钮 hexo-generator-searchdb

GitHub: https://github.com/theme-next/hexo-generator-searchdb  

### Install

```bash
$ yarn add hexo-generator-searchdb
```

### Configure

```yaml
search:
  path: search.xml
  field: post
  content: true
  format: html
```

### Update

```bash
$ yarn upgrade hexo-generator-searchdb
```

## SPA 插件 PJAX for NexT

GitHub: https://github.com/theme-next/theme-next-pjax  

在用户点击网页内链接时，通过 `AJAX` 的方式拉取新页面并渲染，达到 `SPA (Single Page Application)` 的效果。

### Install

```bash
$ cd themes/next
$ git clone https://github.com/theme-next/theme-next-pjax source/lib/pjax
```

But I recommend this

```bash
$ git submodule add https://github.com/theme-next/theme-next-pjax source/lib/pjax
```

### Configure

```yaml
pjax: true
```

### Update

```bash
$ cd themes/next/source/lib/pjax
$ git pull
```

## 网页加载进度条 Progress bar for NexT

GitHub: https://github.com/theme-next/theme-next-pace  

这个插件可以在页面元素加载的时候给页面增加一个进度条提示。

### Install

```bash
$ cd themes/next
$ git clone https://github.com/theme-next/theme-next-pace source/lib/pace
```

I recommend this as well

```bash
$ git submodule add https://github.com/theme-next/theme-next-pace source/lib/pace
```

### Configure

在 `NexT` 配置文件中启用该模块

```yaml
pace:
  enable: true
  # Themes list:
  # big-counter | bounce | barber-shop | center-atom | center-circle | center-radar | center-simple
  # corner-indicator | fill-left | flat-top | flash | loading-bar | mac-osx | material | minimal
  theme: minimal
```

### Update

```bash
$ cd themes/next/source/lib/pace
$ git pull
```

## Sitemap hexo-generator-sitemap

GitHub: https://github.com/hexojs/hexo-generator-sitemap  

### Install 

```bash
$ yarn add hexo-generator-sitemap
```

### Configure

添加设置

```yaml
sitemap:
  path: sitemap.xml
  template: ./path/to/sitemap_template.xml
  rel: false
```

添加 Sitemap 模板文件于你喜欢的位置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  {% for post in posts %}
  <url>
    <loc>{{ post.permalink | uriencode }}</loc>
    {% if post.updated %}
    <lastmod>{{ post.updated.toISOString() }}</lastmod>
    {% elif post.date %}
    <lastmod>{{ post.date.toISOString() }}</lastmod>
    {% endif %}
  </url>
  {% endfor %}
</urlset>
```

对于不想被 Sitemap 包含的文件，可以在头部写入 `sitemap: false` 。

### Update

```bash
$ yarn upgrade hexo-generator-sitemap
```

## RSS hexo-generator-feed

GitHub: https://github.com/hexojs/hexo-generator-feed

### Install

```bash
$ yarn add hexo-generator-feed
```

## Config

```yaml
feed:
  type: atom
  path: atom.xml
  limit: 0
  hub:
  content:
  content_limit: 500
  content_limit_delim: ' '
  order_by: -date
  icon: icon.png
  autodiscovery: true
  template:
```

### Update

```bash
$ yarn upgrade hexo-generator-feed
```

# 最后废话两句

网上很多教程会让你直接修改主题内的文件。这不是不行，但是主题作者已经尽量将配置文件暴露，便于主题升级。

这其中也包括对网站增加插件。举个例子，对于插件 `theme-next-pjax` ， `README.md` 中写的是将插件下载到 `next/source/lib` 下，但我的建议是用 `git submodule add` ，将其作为一整个包插入，也方便以后升级维护。

## NexT 主题解耦合

可以把 `NexT` 的配置文件放置在 `source/_data/next.yml` 下，然后修改该配置文件 `override: true` 即可将设置完全覆盖。对于自己增加的 `CSS` 、 `JS` 等文件，也可以放在 `NexT` 配置文件中提到的对应位置，这对今后的升级和维护有很大帮助。

如果已经在使用 NexT 主题，确保主题文件夹的修改都备份好的情况下，执行 `git rm themes/next` ，从 git 中删除并保留主题原文件。然后执行

```bash
$ git submodule add https://github.com/theme-next/hexo-theme-next themes/next
```

通过增加 submodule 的方式增加主题，并且今后不要再对 `themes/next` 文件夹下的文件做任何修改，否则可能产生一个 `dirty commit` 。

我在通过 `submodule add` 增加 `PJAX, pace` 的时候遇上了下面这个问题

```
YAMLException: end of the stream or a document separator is expected at line 9, column 102:
     ... languages` and other directories:
                                         ^
    at generateError (C:\Projects\Website\wasteland\node_modules\js-yaml\lib\js-yaml\loader.js:167:10)
    ...
```

遇上这种问题，我推测是执行相关命令时， `Hexo` 会对 `source/` 下的所有特定后缀文件进行语法检查。我通过全局搜索含有 `languages and other directories:` 的文件定位到之前添加的 `submodule` 中 `README.md` 存在非法（不符合 YAML 语法检查器）的内容。

添加配置跳过这些文件就可以解决这个问题了。

```yaml
exclude: 
    - lib/**/*.md
```

# References

- 关于@font-face加载前空白(FOIT)的解决方案 - https://juejin.im/post/5a7587d4f265da4e8a31c213  
- Hexo中Gitalk配置使用教程-可能是目前最详细的教程 - https://iochen.com/2018/01/06/use-gitalk-in-hexo/  
