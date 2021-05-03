---
title: 基于hexo + github创建个人博客
date: 2021-05-02 23:07:17
tags:
- 博客
---

# 基于hexo和Github快速创建博客

### 搭建大致流程

1. 创建仓库：<github用户名>.github.io (严格遵守格式)

2. 创建两个分支：master 和 hexo

   master用来保存生成的网页静态文件，而 hexo 用来保存博客的原始文件

3. 设置hexo为默认分支（因为更新博客或者修改博客只需要更新原始文件，然后通过hexo生成静态文件到master分支当中）

4. 把这个仓库克隆到本地

   ```sh
   git clone git@github.com:stephentse9527/stephentse9527.github.io.git
   ```

5. 在本地目录中，依次执行 `npm install hexo`、`hexo init<博客目录> `、`npm install`、`npm install hexo-deployer-git`、`npm install hexo-image-link --save`

   在操作之前，应先把目录中的所有文件移动到其他地方，因为 `hexo init` 需要对应博客目录为空

6. 选择一款博客主题，我这里选的是 [Aurora](https://aurora.tridiamond.tech/zh/)

7. 修改_config.yml中的 deploy 参数，分支应为 master，仓库为我们的 Github Pages

   ```yaml
   deploy:
     type: git
     repository: https://github.com/stephentse9527/stephentse9527.github.io.git
     branch: master
   ```

> 这样搭建后，在Github的 username.github.io 仓库就有两个分支，一个master存放生成的静态网页，一个hexo分支用来存放网站的原始文件

### 日常更新博客流程

##### 在本地更新博客时：

1. 对原始文件进行改动后，依次执行

   ```sh
   git add --all
   git commit -m "message"
   git push origin hexo
   ```

   将改动推送到 Github hexo 分支中

2. 根据原始文件生成静态网页

   ```sh
   // 清空，刷新，部署
   hexo clean && hexo g && hexo d
   ```

##### 在其他电脑维护博客时：

- 直接克隆hexo分支到电脑上，依次执行 npm install hexo、npm install、npm install hexo-deployer-git（不需要hexo init这条指令）
- 然后和普通操作一样改动原始文件后，push即可