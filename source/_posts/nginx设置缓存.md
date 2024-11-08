---
title: nginx设置缓存
date: 2021/6/15 14:28:00   
tags: 
- 优化
categories: 
- 前端
---



问题描述
前端网站采用Vue + Nginx的方式进行生产环境部署。

系统在发布更新次日，收到客户的投诉，发现登录系统之后，出现页面空白问题，刷新几次后显示正常。查看日志发现：

2019/01/07 10:26:01 [error] 19#0: *833 open() "/.../static/js/0.4a66cb25e7f24262c3f6.js" failed (2: No such file or directory), client: XX.XX.XX.XX, server: localhost, request: "GET /static/js/0.4a66cb25e7f24262c3f6.js HTTP/1.1"

问题分析
由于vue在build之后，会重新生成index.html + static资源。从日志判断，【/static/js/0.4a66cb25e7f24262c3f6.js】是上一版本的静态资源。而index.html中get请求获取，跟浏览器缓存有关。

但是查看nginx配置，发现没有配置缓存策略。


    server {
        listen       80 default_server;
        server_name  localhost;
        root         /www/;
        index        index.html index.htm;
        location / {
             try_files $uri $uri/  index.html;
        }
    }

当http头中无缓存配置，那么将使用浏览器默认缓存策略，如下：

先放上结论吧，Chrome和Firefox对js、css之类的文件，在内存中的缓存时长，是： （访问时间 - 该文件的最后修改时间） ÷ 10

假设文件 a.js 最后编辑时间是 2018年12月1号 10点0分0秒；
Chrome的第一次访问时间是 2018年12月1号 12点0分0秒；
第一次访问与文件编辑时间相差2小时，即7200秒，那么缓存时长就是720秒
解决方案
http请求头中，通过cache-control声明缓存策略。

对于index.html采用no-cache策略，客户端使用缓存，但是使用前需要与服务器确认版本状态。
对于static资源，采用客户端缓存。
nginx配置参考：


  location /index.html {
    add_header &quot;Cache-Control&quot; &quot;no-cache&quot;;
  }

  location /static/ {
    access_log none;
    expires 14d;
  }

  location / {
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_pass http://127.0.0.1:8081;
    client_max_body_size  500m;
  }

  location = / {
  }

参考资料
https://blog.csdn.net/youbl/article/details/84879670
https://www.cnblogs.com/feng9exe/p/8083237.html
原文链接：https://my.oschina.net/u/161336/blog/2999036
