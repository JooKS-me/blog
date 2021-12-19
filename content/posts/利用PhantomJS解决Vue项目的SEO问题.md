---
title: "利用PhantomJS解决Vue项目的SEO问题"
date: Fri Feb 19 00:58:49 CST 2021
categories: ["前端"]
author: "JooKS"
authorLink: "https://github.com/JooKS-me"
tags: ["前端"]
draft: false
---

摸鱼的时候进了百度站长平台，对我的新网站进行了爬虫测试，结果发现百度爬虫根本爬不到数据，只有我网站的title。。。

于是百度了一下才发现，这是vue项目的通病。但是办法总比困难多，有好多解决方案。一个是浏览器预渲染，我试了一下，效果并不是很好（可能是因为我不会用），带参数的路由显示不出来。还有一个是ssr服务端渲染，这个看着成本就高，我只有1G内存的小机子估计是顶不住的。

于是，我最终选择了利用PhantomJS解决Vue项目的SEO问题。
https://www.jianshu.com/p/3b72c08cafb2 这篇文章对我帮助很大！！！

首先，是在vue项目中`npm install es6-promise --save`。然后在main.js中加入以下内容：
```javascript
import Promise from 'es6-promise'
Promise.polyfill()
```
然后`npm run build`打包部署服务器。

接着，在服务器上安装node，这里我下载了node官网上的压缩包，然后直接配置了/etc/profile。

然后`npm install pm2 -g`全局安装pm2

wget下载PhantomJS压缩包，解压，放到合适位置，配置环境变量。
我在这里遇到一个坑，报错`吧啦吧啦libssl_conf`之类的。这时需要`export OPENSSL_CONF=/etc/ssl/`，最好是直接在/etc/profile中直接加上，要不然当前这个shell进程结束就没了。参考：https://zhuanlan.zhihu.com/p/157179303

然后`git clone https://github.com/lengziyu/vue-seo-phantomjs.git`
`npm install`
`pm2 start server.js`

最后修改nginx配置文件：
```shell
upstream spider_server {
  server localhost:8081;
}

server {
    listen       80;
    server_name  example.com;
    
    location / {
      proxy_set_header  Host            $host:$proxy_port;
      proxy_set_header  X-Real-IP       $remote_addr;
      proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;

      if ($http_user_agent ~* "Baiduspider|twitterbot|facebookexternalhit|rogerbot|linkedinbot|embedly|quora link preview|showyoubot|outbrain|pinterest|slackbot|vkShare|W3C_Validator|bingbot|Sosospider|Sogou Pic Spider|Googlebot|360Spider") {
        proxy_pass  http://spider_server;
      }
    }
}
```
详细使用参考https://github.com/lengziyu/vue-seo-phantomjs
