---
title: Next主题优化(一)
date: 2019-08-11 18:18:52
updated: 2019-08-11 18:18:52
categories: Hexo
tags: 
 - Hexo
 - Next
---

## 增加底部CNZZ网站访问统计
在自己的网站中增加访问统计,就像我网站底部一样的效果(注:本文撰写时,使用的是Next7.3.0版本)
![](http://mainimage.hi996.com/1565520322384.jpg)
我们可以使用友盟的统计插件,[http://www.umeng.com/](http://www.umeng.com/)
进入网站先注册账号然后根据下列图片进入添加站点。
![](http://mainimage.hi996.com/20190811205836.png)
![](http://mainimage.hi996.com/20190811205928.png)
添加站点，自己搭建的博客，需要统计访问量的网站(这里加入我的博客网站)，然后点击统计代码进入代码页
![](http://mainimage.hi996.com/1565528661814.jpg)
代码页有很多样式，我的是红框的演示，纯文字统计，简洁大方，选择其他样式也可以
![](http://mainimage.hi996.com/1565528815919.jpg)
然后打开文件~/themes/next/layout/_partials/analytics/cnzz-analytics.swig,将友盟的统计代码加进去
![](http://mainimage.hi996.com/1565529138559.jpg)
然后将主题配置文件/next/_config.yml中CNZZ设置为true
![](http://mainimage.hi996.com/1565531405497.jpg)

## 添加侧边栏标签云
标签云的效果见左侧边栏
插件的地址:[https://github.com/MikeCoder/hexo-tag-cloud](https://github.com/MikeCoder/hexo-tag-cloud)
### 安装插件
进入到 hexo 的根目录，在 package.json 中添加依赖: "hexo-tag-cloud": "2.0.*" 操作如下：
``` vim
npm install hexo-tag-cloud@^2.0.* --save
```
### 配置插件
找到侧边栏的代码` /themes/next/layout/_macro/sidebar.swig `,然后添加如下代码
``` html
{% if site.tags.length > 1 %}
<script type="text/javascript" charset="utf-8" src="/js/tagcloud.js"></script>
<script type="text/javascript" charset="utf-8" src="/js/tagcanvas.js"></script>
<div class="widget-wrap">
    <div id="myCanvasContainer" class="widget tagcloud">
        <canvas width="250" height="250" id="resCanvas" style="width=100%">
            {{ list_tags() }}
        </canvas>
    </div>
</div>
{% endif %}
```
代码添加后如下图所示:
![](http://mainimage.hi996.com/1565530763464.jpg)
最后在站点配置文件，即找到 _config.yml配置文件然后在最后添加如下的配置项，可以自定义标签云的字体和颜色，还有突出高亮:
```
# hexo-tag-cloud
tag_cloud:
    textFont: Trebuchet MS, Helvetica
    textColor: '#333'
    textHeight: 25
    outlineColor: '#E2E1D1'
    maxSpeed: 0.1
```
textColor: ‘#333’ 字体颜色
textHeight: 25 字体高度，根据部署的效果调整
maxSpeed: 0.1 文字滚动速度，根据自己喜好调整

##### 本文部分方案参考自:
1. [http://www.aomanhao.top](http://www.aomanhao.top)





