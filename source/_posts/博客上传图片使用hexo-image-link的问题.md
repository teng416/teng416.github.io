---
title: 博客上传图片使用hexo-image-link的问题
date: 2022-03-04 13:50:07
tags: hexo博客上传图片
categories: 博客搭建
description: 使用typora便携markdown插入图片后，图片在typora可正常显示，但在网页中路径存在问题。经过查找网上关于解决此问题的方法，原因主要是markdown中图片的路径格式与asset image中不同，基本所有的帖子都建议使用hexo-image-link插件处理。该插件可以将markdown图片路径自动更改为asset-image格式，但实际操作中仍旧无法解决。本文具体介绍如何处理类似问题。

---

​	使用typora便携markdown插入图片后，图片在typora可正常显示，但在网页中路径存在问题。经过查找网上关于解决此问题的方法，原因主要是markdown中图片的路径格式与asset image中不同，基本所有的帖子都建议使用hexo-image-link插件处理。该插件可以将markdown图片路径自动更改为asset-image格式，但实际操作中图片路径依旧出现了问题。

​	在下载hexo-asset-image包时系统已经在提示存在尚未修复的bug。怀疑可能是bug导致的无法成功转换路径。

```bash
    npm install hexo-asset-image --save
```



解决方法：将hexo-asset-image插件中的index.js文件修改为以下代码再进行部署（记得修改_config.yml文件中的post_asset_folder为True）。

```javascript
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

