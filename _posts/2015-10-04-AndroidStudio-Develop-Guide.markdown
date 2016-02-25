---
layout:     post
title:      "AndroidStudio常用插件推荐"
subtitle:   "AndroidStudio Plugin"
date:       2015-10-04
author:     "YamLee"
header-img: "img/post-bg-universe.jpg"
tags:
    - Android开发
    - AndroidStudio Plugin
    - Guide
---


#AndroidStudio使用指导
>as使用中的一些插件和辅助代码编写的一些建议

## 插件的使用
> 常用插件安装及使用


### SelectorChapek

>SelectorChapek是一款帮助我们快速完成Selector的AndroidStudio插件

1. 安装

>选择Preferences→Plugins→Browse repositories搜索SelectorChapek安装

>下载并在Preferences→Plugins→Install plugin from disk选择安装

2. 使用

>在资源文件夹上右击，如drawable-xhdpi

>选择Generate Android Selectors

>selectors文件会自动生成在drawable文件夹下

3. 命名规则

为了插件的正常运行，资源文件需要正确的命名，该插件支持.png和.9.png文件的识别

文件后缀	    状态

_normal	   `(default_state)`

_pressed	`state_pressed`

_focused	`state_focused`

_disabled	`state_enabled(false)`

_checked	`state_checked`

_selected	`state_selected`

_hovered	`state_hovered`

_checkable	`state_checkable`

_activated	`state_acticated`

_windowfocused	`state_window_focused`

SelectorChapek是一款帮助我们快速完成Selector的AndroidStudio插件


##Android Parcelable plugin

> parcelable code generator 是一款pracelable对象生成工具

##Genymotion

> 一款好用的Android虚拟机

##ideaVim

>Idea环境下的vim编辑环境

##Key Promoter

>提示快捷键提示的插件

##JiMu Mirror

>UI即改即看的一款工具

