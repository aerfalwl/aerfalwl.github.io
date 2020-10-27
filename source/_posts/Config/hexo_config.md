---
title: hexo+github搭建个人博客
date: 2020-10-27 11:00:00
tags:
	- 安装部署
	- 配置
categories: 配置
---

## 步骤
1. 在电脑上搭建博客

   ```she
   hexo init xxx.github.io
   ```

   
2. 修改配置文件_config.yml，添加如下内容：

   ```shell
   deploy:
     type: git
     repository: git@github.com:xxx/youname.github.io.git
     branch: master
   ```

   
3. 提交代码至仓库

   ```shell
   hexo clean # 清空缓存
   hexo generate # 生成静态文件
   hexo deploy # 部署至远程仓库
   ```

4.  创建分支用于保存原配置文件和原博客，因为hexo只会将它生成的静态文件上传，而不会上传相关配置文件和原markdown文件。

   ```shell
   git init
   git checkout -b meta
   ```

5. 之后若有修改，则依次执行如下命令，**注意顺序，且在meta分支执行**

   ```shell
   hexo c && hexo g && hexo d
   git add . && git commit -m "change" && git push origin meta
   ```

## 不同设备同步

执行如下命令

```shell
git clone -b meta https://github.com/XXX/xxx.git
cd xxx.github.io
npm install

# 之后若有修改，则执行
hexo c && hexo g && hexo d
git add . && git commit -m "change" && git push origin meta
```



如果在别的设备上提交了，别忘了在提交之后先pull meta哈。