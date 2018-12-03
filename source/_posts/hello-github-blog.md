---
title: github上搭建hexo博客
comments: true
date: 2018-11-23 20:51:11
tags:
- hexo
categories:
- web
---


打算在github上搭建起hexo博客和gitbook,主要记录一些技术积累.涉及游戏开前后端区块链等.解读一些开源的库.像skynet,pomelo,kbengine,coco2dx,cocos creator,ETH,goworld等.本文记录一下搭建hexo的过程纯属经验之谈.

#### 基本流程

{% asset_img hexo_github.png %}

#### 安装前提

- node.js

  > mac下注意npm对user/local的权限问题
- git  

  > 保证使用ssh和github进行认证测试通过：ssh -T git@github.com.
- 创建github仓库

  > 例如github用户名为gameloses则仓库名为:gameloses.github.io

#### 安装hexo

```
npm install -g hexo-cli
hexo init blog
cd blog
npm install
```
#### 基本配置   

在 _config.yml 中修改大部份的配置  
- 配置部署参数
  ```
  deploy:
    type: git
    repository: git@github.com:gameloses/gameloses.github.io.git
    branch: master
  ```
- 安装部署插件      
  ```
  cd blog
  npm install hexo-deployer-git --save
  ```
- 发布上传博客
  `hexo d -g`
- 常见的其他命令
  ```
  hexo s == hexo server   //启动本地服务
  hexo g == hexo generate //生成
  hexo d == hexo deploy   //发布
  hexo n == hexo new      //新建
  ```

#### 书写文章

`hexo new post "文章名字"`  
使用模板生成文章

```  
title: cocos2dx引擎架构概述
comments: true
date: 2018-11-23 20:51:11
tags:
categories:
```
#### 分类标签
为了使分类标签生效需要生成两个page文件
```
hexo new page categories
hexo new page tags
```
#### 主题配置
一个模仿github样式的主题
```
git clone git@github.com:sabrinaluo/hexo-theme-replica.git themes/replica
```

Set theme: replica in _config.yml (the one in your root folder)     
#### 安装插件
安装rss
`npm install hero-generator-feed`
配置如下：

```
plugin:
- hexo-generator-feed
feed:
type: atom
path: atom.xml
limit: 20
rss: /atom.xml
```

#### 绑定域名

添加WWW和@主机记录，记录类型为CNAME.

 在source目录下创建CNAME文件，文件内容为域名例如：chuangyutime.com

#### QA     

1. mac下node安装好之后使用npm安装全局包会出现usr/local目录权限读写问题？
`sudo chown -R $USER /usr/local`
修改权限之后使用ls -l /usr/local 查看权限
2. 分类标签404？
需要创建两个page categories、tags
3. vscode 编写markdown回退文本之后以后存在特殊的bs字符问题？    
显示隐藏字符 `"editor.renderControlCharacters": true`   
安装插件：`Remove backspace control character`  
开启设置：`"editor.formatOnType": true 在被设定的情况下，进行变换时;输入时启动`

