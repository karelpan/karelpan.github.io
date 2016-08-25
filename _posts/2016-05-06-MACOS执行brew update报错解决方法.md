---
layout: post
title: 解决MAC OS新系统更新后，执行``brew update``报错
---

 {{ page.title }}
================

<p class="meta">06 May 2016 - Su Zhou</p>

当你执行 ``brew update``时，新系统你会遇到如下报错画面

```
/System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/2.0.0/rubygems/core_ext/kernel_require.rb:55:in `require': cannot load such file -- mach (LoadError)
 from /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/2.0.0/rubygems/core_ext/kernel_require.rb:55:in `require'
 from /usr/local/Library/Homebrew/extend/pathname.rb:2:in `<top (required)>'
 from /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/2.0.0/rubygems/core_ext/kernel_require.rb:55:in `require'
 from /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/2.0.0/rubygems/core_ext/kernel_require.rb:55:in `require'
 from /usr/local/Library/Homebrew/global.rb:3:in `<top (required)>'
 from /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/2.0.0/rubygems/core_ext/kernel_require.rb:55:in `require'
 from /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/ruby/2.0.0/rubygems/core_ext/kernel_require.rb:55:in `require'
 from /usr/local/Library/brew.rb:15:in `<main>'
```

通过如下方式可以解决此问题

```bash
// Open a terminal and type:
$ sudo chown -R $(whoami):admin /usr/local 
$ cd /usr/local 
$ git reset --hard 
$ git clean -df
$ brew update
```
