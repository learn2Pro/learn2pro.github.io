---
title: HEXO+GITHUB搭建个人博客
date: 2017-01-24 17:24:00
categories:
- Essay
tags:
- default
---

在github上搭建[博客](http://learn2pro.tech)一般使用Jekyll static blogs的方式，非常方便快捷，但是有个缺点便是切换theme很不方便；用Jekyll相对普通人来说已经足够，但是如果需要强大的模版，一些可扩展的东西，就会比较麻烦，需要对前端有一定的技术背景。
所以出现了
* Hexo :+1:
```
What is Hexo?Hexo is a fast, simple and powerful blog framework. You write posts in Markdown (or other languages) and Hexo generates static files with a beautiful theme in seconds.
```
Hexo ＝ Node.js + MarkDown
Hexo切换模板非常方便只需要修改配置文件重新generate一下，Hexo官方文档[Document](https://hexo.io/docs/)
安装Hexo之前需要安装node和git：

 * Node.js
 * Git

如果你的电脑已经安装好了node和git,使用npm安装：

```bash
$ npm install -g hexo-cli
```
然后去挑选喜欢hexo模板[themes](https://hexo.io/themes/)开始搭建个人博客了,或者github上面搜索hexo themes有很多高star的模板，[本文的博客](http://learn2pro.tech)
采用的是[litten](http://litten.me),非常感谢;

```bash
$ mkdir <foldser>
$ hexo init <folder>
$ cd <folder>
$ npm install
当我们初始化完成后我们的目录结构如下
.
├── _config.yml
├── package.json
├── scaffolds
├── source
|   ├── _drafts
|   └── _posts
└── themes
```

<!-- more -->
主要的配置是放在_config.yml中具体每一项的意义可以参考[configuration](https://hexo.io/docs/configuration.html)
source目录存放的是文章的内容
![image](https://cloud.githubusercontent.com/assets/17799557/22239845/3c99f700-e253-11e6-98cd-0e800f193ca2.png)
```bash
$ hexo new [layout] <title>
```
创建一个新的文章*.md，md使用MarkDown语法,推荐使用markdown-editor,md是一种可以很方便编辑转换为html的格式
然后是对自己的博客做扩展添加评论和站长统计，baidu和google都会介绍

 * Analytics
 
![image](https://cloud.githubusercontent.com/assets/17799557/22240151/b794c308-e254-11e6-96b7-ebdb55fa59de.png)

有的theme/_config.yml会有此配置项，将track_id配置进去即可，如果没有百度的 ，需要手动将baidu analytics的js代码加进
```bash
├── .deploy_git
|   ├── index.html
```
里面，过几分钟就能看到[数据](https://analytics.google.com);

 * Comment
 
 评论使用的是duoshuo的评论框，去[注册](http://duoshuo.com/)账号拿到short_key写入到theme/_config.yml中
 
 ```bash
  duoshuo: short-key
```
如果没有此配置项手动加入到目录下

```bash
├── .deploy_git
|   ├── index.html
```

然后就需要生成我们的代码了，到根目录

```bash
$ 
```

配置根目录的config如下

![image](https://cloud.githubusercontent.com/assets/17799557/22240571/b71951c6-e256-11e6-908b-96c3f8f48e9b.png)

配置仓库地址，此处的问题是github屏蔽了baidu的爬虫，所有如果你的博客想被baidu索引到可以使用其他仓库地址，比如[coding](https://coding.net/git)
```bash
$ npm install hexo-deployer-git --save //如果遇到ERROR Dehexo generateployer not found: git，安装下
$ hexo clean && hexo deploy
```

 * 域名
 
 博客需要自己的专属域名，可以去godaddy或者万网上买一个，然后解析域名到github的[ip上](https://help.github.com/articles/setting-up-an-apex-domain/#configuring-a-records-with-your-dns-provider) 192.30.252.153/192.30.252.154
 需要创建一个CNAME的文件来告诉username.github.io你的域名是什么
 
 ```bash
├── source
|   ├──CNAME
```
最后push到github上博客就可以生成了