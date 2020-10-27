---
title: hexo 插入图片
date: 2020-10-28 00:09:12
tags:
	- 安装部署
	- 配置
categories: 配置
---

本文主要介绍按照相对路径在hexo中插入图片的方法。

## 主要步骤

1. 安装插件

   ```shell
   npm install hexo-asset-image --save
   ```

2. 修改配置文件_config.yml，将如下选项该为true：

   ```shell
   post_asset_foler: true
   ```
   
3. 更改文件内容：node_modules/hexo-asset-image/index.js为

   ```js
   'use strict';
   var cheerio = require('cheerio');
   
   // http://stackoverflow.com/questions/14480345/how-to-get-the-nth-occurrence-in-a-string
   function getPosition(str, m, i) {
     return str.split(m, i).join(m).length;
   }
   
   var version = String(hexo.version).split('.');
   hexo.extend.filter.register('after_post_render', function(data){
     var config = hexo.config;
     if(config.post_asset_folder){
           var link = data.permalink;
       if(version.length > 0 && Number(version[0]) == 3)
          var beginPos = getPosition(link, '/', 1) + 1;
       else
          var beginPos = getPosition(link, '/', 3) + 1;
       // In hexo 3.1.1, the permalink of "about" page is like ".../about/index.html".
       var endPos = link.lastIndexOf('/') + 1;
       link = link.substring(beginPos, endPos);
   
       var toprocess = ['excerpt', 'more', 'content'];
       for(var i = 0; i < toprocess.length; i++){
         var key = toprocess[i];
    
         var $ = cheerio.load(data[key], {
           ignoreWhitespace: false,
           xmlMode: false,
           lowerCaseTags: false,
           decodeEntities: false
         });
   
         $('img').each(function(){
           if ($(this).attr('src')){
               // For windows style path, we replace '\' to '/'.
               var src = $(this).attr('src').replace('\\', '/');
               if(!/http[s]*.*|\/\/.*/.test(src) &&
                  !/^\s*\//.test(src)) {
                 // For "about" page, the first part of "src" can't be removed.
                 // In addition, to support multi-level local directory.
                 var linkArray = link.split('/').filter(function(elem){
                   return elem != '';
                 });
                 var srcArray = src.split('/').filter(function(elem){
                   return elem != '' && elem != '.';
                 });
                 if(srcArray.length > 1)
                   srcArray.shift();
                 src = srcArray.join('/');
                 $(this).attr('src', config.root + link + src);
                 console.info&&console.info("update link as:-->"+config.root + link + src);
               }
           }else{
               console.info&&console.info("no src attr, skipped...");
               console.info&&console.info($(this));
           }
         });
         data[key] = $.html();
       }
     }
   });
   ```

   

4. 至此配置完成，之后若hexo new post photo，则会在source/_posts文件夹下生成photo.md和photo文件夹，用户也可以在手动创建md文件的时候，创建对应名字的文件夹。

5. 插入图片，假设photo文件夹下有如下文件图片：test.png，则用户可以在md文件中按照如下两种方式引用该图片。

   ```txt
   ![描述](test.png) # 
   ![描述](photo/test.png) # 该种方式在Typora中可以直接显示图片，推荐该种方式。
   ```