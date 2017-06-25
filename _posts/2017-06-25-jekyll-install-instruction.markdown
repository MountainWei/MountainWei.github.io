---
layout:     post
title:      "博客评论系统更换记"
subtitle:   "jekyll安装和网易云跟帖安装"
date:       2017-06-25 11:55:00
author:     "Liuv"
header-img: "img/post-bg-2015-12-05.jpg"
tags:
    - blog
    - jekyll
    - ruby
---

> 多说在今年的六一儿童节倒下了，呜呼哀哉。

一年没有更新博客了，被找工作、写毕业论文这些事情弄得无法静下心来。终于等到最近一切尘埃落地，准备打开博客开始整理这一年的收获，却发现博客的评论系统down掉了。经过排查发现多说竟然在今年六一儿童节宣布倒闭，一声叹息后，只能更换社交评论系统，作为网易的忠实用户，最终选择了网易云跟帖作为博客的评论系统。接下来就开始了折腾，主要分为安装jekyll运行环境和注册并添加网易云跟帖js代码。 
## 安装jekyll3
jekyll3安装之前需要安装以下工具：
 - Ruby，要求版本在2.1以上。
 - RubyGems。
 - GCC和Make工具。

 ==注：jekyll2时代还需要安装Node.js。==

### Ruby安装
根据操作系统的不同，Ruby主要有以下3种安装方法：
 - 对于Linux系统，可以使用系统自带的包管理系统或者第三方的安装工具（官方推荐rbenv和RVM）
 - 对于OS X系统，使用第三方的安装工具（官方推荐rbenv和RVM）
 - 对于windows系统，使用RubyInstaller进行安装
我当前的系统为ubuntu16.04，使用apt安装ruby后，发现版本为1.9，低于要求的2.1.于是只能使用第三方的安装工具。我首先选择的是RVM，根据官网上的文档添加key并安装后，出现了奇怪的permission denied错误，一番排查未果后果断弃之，不在这上面浪费时间。选择官方推荐的另一个安装工具rbenv，然后一路顺利成功安装，方法如下：
1. 安装rbenv。rbenv本身是一个ruby的管理工具，能够管理系统中不同版本的ruby。
```
git clone https://github.com/rbenv/rbenv.git ~/.rbenv
echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(rbenv init -)"' >> ~/.bashrc
source ~/.bashrc
type rbenv | head -1
```
若最后的输出结果为“rbenv is a function.”则说明rbenv安装成功。
2. 安装ruby-build插件。rbenv只是管理工具，需要借助于ruby-build这个插件来编译和安装ruby。
```
git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build
echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```
网上有些教程说还需要安装rbenv-gem-rehash插件，其实是多余的步骤。因为这个插件已经被官方废弃，插件提供的功能现在已经包含在rbenv core中。
3. 安装ruby、gems。这里我们安装目前最新版本2.4.1。
```
rbenv install 2.4.1
rbenv global 2.4.1
ruby -v
gem install bundle
```

### 安装jekyll
1. 安装jekyll
```
gem install jekyll
```
2. 检查jekyll是否安装成功。
```
jekyll -v
jekyll new test-site
cd test-site
jekyll serve
```
然后访问localhost:4000网址看看能否顺利打开。

## 更换网易云跟帖系统
1. 注册网易云跟帖网站并进入到后台。
2. 设置站点信息。
3. 自定义评论系统的皮肤和提示信息。
4. 获取WEB代码并添加到博客的_layouts/post.html文件中。

终于换好了，希望网易云跟帖能多坚持几年，只是我之前的那些评论数据再也找不到了。
