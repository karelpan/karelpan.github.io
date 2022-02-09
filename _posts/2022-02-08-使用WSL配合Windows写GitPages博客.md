layout: post
title: 2022-02-08-使用WSL配合Windows写GitPages博客
uuid: 78bf8738-c61c-4db9-ab57-eb547d007949
----

{{ page.title }}
================

首先是安装 wsl，我选择的是 Ubuntu，在你的Windows下，Win键+ x键 可打开一个系统工具列表，选中 “PowerShell（管理员）”并鼠标点击，则会出现一个具有管理源权限的PowerShell 终端界面
\\
 * 使用 “PowerShell（管理员）”，是因为操作 wsl 需要管理员权限 （本内容同样可支持 WSL1 + WSL2）
接着在 PowerShell 命令行框中键入
```bash
wsl --install -d Ubuntu
```
安装完 wsl 以后，我们首先要将 apt 源更换为 aisa区的，以提高更新速度，比如你可以选用清华大学的源\\
接着我们安装node，首先打开 Ubuntu （直接可以在应用列表找到已经安装的 WSL Ubuntu）
```bash
# 在 Ubunut 的终端下安装 Node 和ruby
sudo apt update
sudo apt upgrade -y
sudo apt node -y 
sudo apt npm -y
sudo apt ruby -y
sudo npm install bundler
```

然后你需要配置好你的 GitHub 的 gitpages 用的 项目的 免登录拉git 的key，具体可以到 github 看怎么操作\\
接着我们需要进行拉下  gitpages，我的gitpages 使用了 jekyll
```bash
cd ~ 
mkdir -p git/blog
git clone git@github.com:xxx/xxx.github.io.git
cd xxx.github.io
bundle install --path vendors/bundle
```
等待安装完成，开始使用 jekyll 写博客吧, 下面先启动 jekyll 测试服务器

```bash
bundle exec jekyll serve --livereload
```  
   
# 如何在windows 上使用 vscode 编辑呢？
安装好windows 版本的 vscode，启动编辑器，在 “文件”下选择“打开文件夹”，会立即弹出一个文件夹框；接着在文件夹框的地址栏中输入 ```\\wsl$``` 可以看到你的 wsl 虚拟机磁盘目录\\
进入直接一层层找到你在 ubunut 下设置的 blog 文件目录，比如 ```\\wsl$\Ubuntu\home\xxxx\xxx\xxx.github.io```  

至此 环境的配置就ok了



