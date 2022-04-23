---
title: Hexo博客
date: 2022-04-20
toc: true #是否显示文章目录
categories: "生活" #分类
tags:   #标签
    - 博客
---
 使用git pages和hexo来搭建自己的博客，是一件幸福的事。

1. 安装nodejs https://nodejs.org/en/ 下载安装pkg
   安装nodejs的时候一开始安装的16，在install moudle的时候报错，删除后重新安装14，才可以使用，
   这点需要注意。

2. npm 安装hexo
   npm配置国内的访问地址 
    ```bash
       npm config set registry https://registry.npm.taobao.org 
        sudo npm install -g hexo-cli 
        npm install hexo -g #安装Hexo
        hexo n "我的博客" == hexo new "我的博客" #新建文章
        hexo g == hexo generate #生成
        hexo s == hexo server #启动服务预览
        hexo d == hexo deploy #部署

        hexo server #Hexo会监视文件变动并自动更新，无须重启服务器
        hexo server -s #静态模式
    ```

3. 需要重新安装主题，主题没有默认提交到github
   ```bash
          git clone https://github.com/tufu9441/maupassant-hexo.git themes/maupassant
          npm install hexo-renderer-pug --save
          npm install hexo-renderer-sass --save
   ```

4. 图床
   我还是使用github,先把img上传到github上去，然后在md文档中使用。可以参考下面的例子：
   ```bash
   ![老边沟枫叶](https://raw.githubusercontent.com/student2028/blog_markdown/master/img/fengye1.jpeg)
   ```
5. hexo d执行的时候，需要提前把本机的.ssh中的pub配置到github上，同时需要执行下面两行命令。
   ```bash
     git config --global user.email "dailyyaoxh@126.com"
     git config --global user.name "student2028"
   ```
