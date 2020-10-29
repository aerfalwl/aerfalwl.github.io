---
title: hexo 博客被google收录
date: 2020-10-27 22:57:12
tags:
	- 安装部署
	- 配置
categories: 配置
---

## 步骤
1. 生成站点地图

   ```shell
   npm install hexo-generator-sitemap --save
   ```

2. 修改配置文件_config.yml，添加如下内容：

   ```shell
   urlgoogle: https://xxx.github.io/ 
   sitemap:
     path: sitemap.xml
   ```

3. 修改_config.xml，设置url为你的github.io

   ```xml
   url: https://xxx.github.io
   ```

   

4. 添加站点，登录[Google 网站站长](https://www.google.com/webmasters/)，进入```Search Console```，进入如下页面

   ![首页](hexo_google_site/first.png)

5. 选择网址前缀，输入https://xxx.github.io

6. 之后，下载Google验证文件，放在```theme/next/source```目录中。

7. 生成robots.xt文件，在hexo的source目录下，放入如下内容：

   ```txt
   User-agent: *
   Allow: /
   Allow: /home/
   Allow: /archives/
   Allow: /categories/
   Allow: /tags/
   
   Disallow: /js/
   Disallow: /css/
   Disallow: /fonts/
   
   Sitemap: https://aerfalwl.github.io/sitemap.xml
   ```

   

8. 重新生成和部署

   ```shell
   hexo clean && hexo generate && hexo deploy
   ```

9. 部署完成之后，进行验证即可，若操作无误，便会验证成功。

10. 添加站点地图：

   ![添加站点地图](hexo_google_site/add_sitemap.png)

11. 为了加快Google扫描网站的速度，可以通过以下方式建立索引：

    - 在浏览器中输入https://www.google.com/ping?sitemap=https://aerfalwl.github.io/sitemap.xml要求Google给网站建立索引；

12. 等待几分钟，在Google中搜索site:aerfalwl.github.io检验是否能看到网站内容。

