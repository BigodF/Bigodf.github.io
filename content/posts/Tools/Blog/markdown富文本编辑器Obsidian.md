---
title: Obsidian
date: 2025-11-28T16:56:24+0800
lastmod: 2025-11-28T17:43:09+0800
author:
- Bigodf
tags:
- Tools
- Obsidian
- 富文本编辑器
description: Obsidian
summary: ''
weight: null
slug: ''
draft: false
comments: true
showToc: true
TocOpen: true
autonumbering: true
hidemeta: false
disableShare: true
searchHidden: false
showbreadcrumbs: true
mermaid: true
cover:
  image: ''
  caption: ''
  alt: ''
  relative: false


---

本文记录一下我是如何找到一款适合自己的md富文本编辑器。
我对富文本编辑器的需求：
1. 免费！最好开源，跨平台。
2. WYSIWYG（所见即所得）。
3. 有序列表可以自动缩进、自动编号（删除时也可以自动编号）。
4. 支持粘贴图片。
5. 支持编辑表格。
6. 代码块能够折叠，快内代码能够自动换行。
7. 支持页面全宽。

VScode是我一直在使用的编辑器，所以第一选择是找一个好用的VSCode插件，找了一些，都不是很好用，唯一好用的收费。
- VScode自带的md编辑器，不是WYSIWYG。
- Markdown Editor（zaaack.markdown-editor）。使用体检不是很好，代码块的WYSIWYG分为上下两个部分，上面编辑，下面显示，设计的很别扭。
- Markdown Editor（adamerose.markdown-wysiwyg）。体验也不是很好。
- Mark Sharp（jonathan-yeung.mark-sharp）。猜测很完美，但是免费版不支持粘贴图片、编辑表格。买断88￥。

最终没有找到合适的插件，转向其它富文本APP。
- Typora。收费，放弃，14$。
- MarkText。Typora的开源版本，22年就不在更新了，不支持代码自动换行，打开文件夹时，文件夹的打开、关闭图标状态很难区分。github上有人维护了一个分支，但是没有mac版本，放弃。
- Zettlr。莫名难用，具体忘记了，好像是针对学术方向。
- Obsidian。除了代码换行没有，其它完美，而且支持插件、支持自定义css。第三方插件可以支持代码换行。

## Obsidian配置
### 安装插件
首先介绍插件的安装方式：
1. 在app的插件中心直接下载安装，需要翻墙。
2. 本地下载，然后安装。

本地安装方式：
1. 找到插件的git仓库release版本，下载以下三个文件：
	1. main.js
	2. manifest.json
	3. styles.css
2. 在workspace的文件夹中，找到.obsidian/plugins文件夹（我是先在app上安装了一个插件，直接就有了这个目录，如果没有安装过插件的话试试新建？），然后把新建一个空的插件文件夹，把上面三个文件放到文件夹下即可。
### 代码块折叠插件
Code Styler
使用方式：
````bash
```python fold
<some code>
```
````
