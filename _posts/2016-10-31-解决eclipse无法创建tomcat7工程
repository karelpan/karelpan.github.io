---
layout: post
title: 解决eclipse无法创建 tomcat7 工程
uuid: c4e77e4e-a003-11e6-a5d0-005056359bbd
---

 {{ page.title }}
================

<p class="meta">31 Oct 2016 - Su Zhou</p>

# 当你正常删除 已经创建的 tomcat 工程，并且删除Srever View 中的 Server 配置， 有的时候，你会发现莫名其妙，tomcatX 不可以再被创建

So，你只需要输入以下一句话

```bash
# 1. 到 eclipse 的 workspace 目录下， 进入 .metadata 文件夹（这个文件夹默认是被隐藏的）
#    linux 下 `cd workspace/.metadata`，
#    windows 下 Powershell ， `cd workspace/.metadata`, 后 `start .` 打开文件夹
# 2. 在次文件夹下查找文件 "org.eclipse.wst.server.core.prefs" , "org.eclipse.jst.server.tomcat.core.prefs"
#    linux 下 `find . | grep "org.eclipse.wst.server.core.prefs"; find . | grep "org.eclipse.jst.server.tomcat.core.prefs"`
#    windows 下 UI 界面搜索 
# 3. 删了这两个文件
#    linux 下
rm org.eclipse.wst.server.core.prefs org.eclipse.jst.server.tomcat.core.prefs

#    windows 删除文件非常直观，就不说了

```