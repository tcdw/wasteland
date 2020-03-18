---
title: 使用 LaTeX 完成你的毕业论文
date: 2020-03-17 14:02:47
categories:
- Talk
tags:
- LaTeX
- Paper
- Thesis
---

本文使用 **武汉大学** 毕业论文模板。

# 准备写论文

可喜可贺，我终于要开始动笔我的毕业论文了。虽然论文相关的实验和理论准备已经就绪了很长一段时间，但是就是提不起精神写文章。这周其他突发堆积的事情逐渐清空，我准备忙里偷闲，开始写毕业论文。

某天，一同学在群里发了一份 [武汉大学毕业论文 LaTeX 模板](https://github.com/mtobeiyf/whu-thesis/)，我一看，巧了，老朋友写的 `repository` ，于是正好近期开工，把它下载下来用。

# 环境配置

> 本文基础环境是 Windows 10，使用的 IDE 为 **VSCode**，Linux、Mac 配置方式类似

## Perl

首先我们需要安装 Windows 下的 `Perl` 依赖。前往官网 https://www.perl.org/get.html 我发现，有 `ActiveState Perl` 和 `Strawberry Perl` 两种版本的 `Perl` ，具体使用起来差不多，所以我选择了下载起来相对方便的 `草莓 Perl` http://strawberryperl.com/ 。

## LaTeX

Windows 下有两种广泛使用的 LaTeX 版本，分别是 `TeX Live` 和 `MiKTeX` 。 `TeX Live` 安装体积高达 **6GB** ，我立刻放弃，选择了安装体积较小，后续使用中**按需下载**的 `MiKTeX` https://miktex.org/download 。建议在安装时安装给**当前用户**，可以避免很多麻烦。

<!--more-->

{% asset_img Install-for-current-user.jpg Install for current user %}

同时，我们需要手工指定镜像。根据我的使用情况， `HIT` 的源中包含了很多不可用的文件，会拖慢下载速度。打开 `MiKTeX` 客户端，进入 `Settings` - `General` - `Package Installation` 更换源。推荐更换为 `清华源` 。

{% asset_img Change-TeX-mirror.jpg Change TeX mirror %}

## VSCode

首先需要在 VSCode 中安装依赖 `LaTeX Workshop` 。

VSCode 的配置文件在上述模板中包含。但是我建议对配置文件进行一定的修改，使用更方便的安装命令。并将工程和 pdf 文件构建于 `build/` 文件夹下，保持根目录干净。

可以在 https://gist.github.com/HoshinoTouko/a2332b1756996d5e9c71605d0ff7591a 获取 `.vscode/settings.json` 和 `.latexmkrc` ，下载后放置于对应目录即可工作。

## 手工执行

也可以通过手工执行的方式对工程进行编译。下载上文的 `.latexmkrc` 放置于根目录，如果无法访问 gist ，也可以复制下方文件。然后确认自己的 `PowerShell` 工作于 `UTF-8` 编码下。确认 `PowerShell` 编码可以执行 `chcp` 命令。

```.latexmkrc .latexmkrc
$pdflatex=q/xelatex -synctex=1 %O %S/
```

```ps 确认 PowerShell 编码
PS C:\Research\undergraduate-thesis> chcp
Active code page: 65001
```

如果 `Active code page` 是 65001 ，则该 PowerShell 运行于 `UTF-8` 编码下。如果不是，执行 `chcp 65001` 即可。如需更改默认 PowerShell 编码，可以通过以下方式。

{% asset_img Change-ps-to-UTF8.jpg Change ps to utf-8 %}

执行 

```ps 使用 latexmk 调用 xelatex 编译 LaTeX 文档
latexmk -xelatex -pdf -synctex=1 -interaction=nonstopmode -file-line-error -shell-escape -outdir=build ./main
```

即可在 `build/` 文件夹下完成工程构建。

# 写在最后

其实我本来是想用 `overleaf` 的，但是觉得学位论文这种东西，在线保存有一定风险，还是握在自己手里比较好。所以我就捣鼓了几个小时，把环境都部署好跑起来了。本文提供的样例是武汉大学的，在 GitHub 或者 Google 根据关键词搜索，即可找到别的学校的同学制作的相关模板。例如

- 浙江大学毕业设计/论文 LaTeX 模板 - https://github.com/TheNetAdmin/zjuthesis
  包含本科生、硕士生与博士生模板，以及英文硕博士模板
- Tsinghua University Thesis LaTeX Template - https://github.com/xueruini/thuthesis
- fduthesis - https://github.com/stone-zeng/fduthesis
- 上海交通大学 XeLaTeX 学位论文及课程论文模板 - https://github.com/sjtug/SJTUThesis
- ...

希望大家度过一个特殊又充实的毕业论文假期

# References

- 武汉大学毕业论文 LaTeX 模版 2019 - https://github.com/mtobeiyf/whu-thesis
- How to make LaTeXmk work with XeLaTeX and biber - https://tex.stackexchange.com/questions/27450/how-to-make-latexmk-work-with-xelatex-and-biber
- Using Latexmk - https://mg.readthedocs.io/latexmk.html
- How do I run bibtex after using the -output-directory flag with pdflatex, when files are included from subdirectories? - https://tex.stackexchange.com/questions/12686/how-do-i-run-bibtex-after-using-the-output-directory-flag-with-pdflatex-when-f
- CTAN 镜像使用帮助 - https://mirror.tuna.tsinghua.edu.cn/help/CTAN/
- LaTeX-Workshop GitHub - https://github.com/James-Yu/LaTeX-Workshop
