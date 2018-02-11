---
title: What is Hexo?
date: 2017-02-23 11:31:11
tags:
categories: Hexo
---
## 1. Hexo简介

Hexo 是一款基于 Node.js 的静态博客框架。Hexo 使用 Markdown 解析文章，用户在本地安装Hexo并进行写作，通过一条命令，Hexo即可利用靓丽的主题自动生成静态网页。
参考：Hexo Github地址     Hexo帮助文档(https://hexo.io/zh-cn/docs/)
## 2. 如何使用Hexo搭建自己的博客

### uninstall old version of nodejs & npm

``` bash
$ sudo apt-get purge nodejs npm
```

### update gcc g++

``` bash
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt-get update
sudo apt-get install gcc-4.9
sudo apt-get install g++-4.9

update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-4.9 100
update-alternatives --install /usr/bin/gcc gcc /usr/bin/g++-4.9 100
```

### install nodejs

``` bash
$ wget https://nodejs.org/dist/v6.9.5/node-v6.9.5.tar.gz
$ tar xvf node-v6.9.5.tar.gz
$ ./configure
$ make 
$ make install
```

### install npm

``` bash
$ sudo apt-get install npm
```

### check res

``` bash
$ node -v
v6.9.5

$ npm -v
3.10.10
```

### 安装并初始化Hexo

``` bash
$ npm install -g hexo-cli
$ hexo init
```
安装完成后，指定文件夹的目录如下：
``` bash
1 ├── _config.yml
2 ├── package.json
3 ├── scaffolds
4 ├── source
5 |   ├── _drafts
6 |   └── _posts
7 └── themes
```
其中_config.yml文件用于存放网站的配置信息，你可以在此配置大部分的参数；scaffolds是存放模板的文件夹，当新建文章时，Hexo 会根据scaffold来建立文件；source是资源文件夹，用于存放用户资源，themes是主题文件夹，存放博客主题，Hexo 会根据主题来生成静态页面。


``` bash
$ hexo new "My New Post"
```

### Run server

``` bash
$ hexo server
```

### Generate static files

``` bash
$ hexo generate
```

### Deploy to remote sites

``` bash
$ hexo deploy
```
