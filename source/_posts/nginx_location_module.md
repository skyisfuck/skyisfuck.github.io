---
title: nginx_location_module
date: 2018-10-05 22:03:46
tags: nginx
---
### 原文：https://moonbingbing.gitbooks.io/openresty-best-practices/content/ngx/nginx_local_pcre.html

#### 语法规则
><font color=green>  location [=|~|~*|^~] /uri/ { … }</font>

#### 五种匹配方式
```
=	开头表示精确匹配
^~	开头表示 uri 以某个常规字符串开头，理解为匹配 url 路径即可。nginx 不对 url 做编码，因此请求为/static/20%/aa，可以被规则^~ /static/ /aa匹配到（注意是空格）
~	开头表示区分大小写的正则匹配
~*	开头表示不区分大小写的正则匹配
/	通用匹配，任何请求都会匹配到
```

多个 location 配置的情况下匹配顺序为（参考资料而来，还未实际验证，试试就知道了，不必拘泥，仅供参考）:
- 首先匹配 =
- 其次匹配 ^~
- 其次是按文件中顺序的正则匹配
- 最后是交给 / 通用匹配
- 当有匹配成功时候，停止匹配，按当前匹配规则处理请求
例子，有如下匹配规则：

```
location = / {
   #规则A
}
location = /login {
   #规则B
}
location ^~ /static/ {
   #规则C
}
location ~ \.(gif|jpg|png|js|css)$ {
   #规则D
}
location ~* \.png$ {
   #规则E
}
location / {
   #规则F
}
```

那么产生的效果如下：
- 访问根目录 /， 比如 `http://localhost/` 将匹配规则 A
- 访问 `http://localhost/login` 将匹配规则 B，`http://localhost/register` 则匹配规则 F
- 访问 `http://localhost/static/a.html` 将匹配规则 C
- 访问 `http://localhost/a.gif, http://localhost/b.jpg` 将匹配规则 D 和规则 E，但是规则 D 顺序优先，规则 E 不起作用，而 `http://localhost/static/c.png` 则优先匹配到规则 C访问 `http://localhost/a.PNG` 则匹配规则 E，而不会匹配规则 D，因为规则 E 不区分大小写。
- 访问 `http://localhost/category/id/1111` 则最终匹配到规则 F，因为以上规则都不匹配，这个时候应该是 nginx 转发请求给后端应用服务器，比如 FastCGI（php），tomcat（jsp），nginx 作为反向代理服务器存在。
所以实际使用中，笔者觉得至少有三个匹配规则定义，如下：

```
# 直接匹配网站根，通过域名访问网站首页比较频繁，使用这个会加速处理，官网如是说。
# 这里是直接转发给后端应用服务器了，也可以是一个静态首页
# 第一个必选规则
location = / {
    proxy_pass http://tomcat:8080/index
}

# 第二个必选规则是处理静态文件请求，这是 nginx 作为 http 服务器的强项
# 有两种配置模式，目录匹配或后缀匹配，任选其一或搭配使用
location ^~ /static/ {
    root /webroot/static/;
}
location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
    root /webroot/res/;
}

# 第三个规则就是通用规则，用来转发动态请求到后端应用服务器
# 非静态文件请求就默认是动态请求，自己根据实际把握
# 毕竟目前的一些框架的流行，带.php、.jsp后缀的情况很少了
location / {
    proxy_pass http://tomcat:8080/
}

```


### ReWrite语法
- last – 基本上都用这个 Flag
- break – 中止 Rewirte，不再继续匹配
- redirect – 返回临时重定向的 HTTP 状态 302
- permanent – 返回永久重定向的 HTTP 状态 301

1、下面是可以用来判断的表达式：
```
-f 和 !-f 用来判断是否存在文件
-d 和 !-d 用来判断是否存在目录
-e 和 !-e 用来判断是否存在文件或目录
-x 和 !-x 用来判断文件是否可执行
```
2、下面是可以用作判断的全局变量
```
例：http://localhost:88/test1/test2/test.php?k=v
$host：localhost
$server_port：88
$request_uri：/test1/test2/test.php?k=v
$document_uri：/test1/test2/test.php
$document_root：D:\nginx/html
$request_filename：D:\nginx/html/test1/test2/test.ph
```

### Redirect 语法
```
server {
    listen 80;
    server_name start.igrow.cn;
    index index.html index.php;
    root html;
    if ($http_host !~ "^star\.igrow\.cn$") {
        rewrite ^(.*) http://star.igrow.cn$1 redirect;
    }
}
```
### 防盗链
```
location ~* \.(gif|jpg|swf)$ {
    valid_referers none blocked start.igrow.cn sta.igrow.cn;
    if ($invalid_referer) {
       rewrite ^/ http://$host/logo.png;
    }
}
```
### 根据文件类型设置过期时间
```
location ~* \.(js|css|jpg|jpeg|gif|png|swf)$ {
    if (-f $request_filename) {
        expires 1h;
        break;
    }
}
```
### 禁止访问某个目录
```
location ~* \.(txt|doc)${
    root /data/www/wwwroot/linuxtone/test;
    deny all;
}
```
### 显示客户端真实ip
```
nginx的配置
location \ {
    proxy_set_header   X-Real-IP  $remote_addr;
    proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for; 
}
X-Forwarded-For和X-Real-IP的区别是，如果请求时已带了X-Forwarded-For，则nginx追加方式，这样可以通过它显示转发的轨迹。
当然请求时完全可以构造假的X-Forwarded-For，在配置文件打开了X-Real-IP及编译指定了--with-http_realip_module时，环境变量HTTP_X_REAL_IP总是为真实的客户端IP。
后端server配置
httpd 
LogFormat "%h %l %u %t /"%r/" %>s %b /"%{Referer}i/" /"%{User-Agent}i/"" combined
把其中的%h 换成 %{X-Real-IP}i
```

### proxy_cache_path
```
http{
    ......
    proxy_cache_path /data/nginx/tmp-test levels=1:2 keys_zone=tmp-test:100m inactive=7d max_size=1000g;
# 代码说明：
# proxy_cache_path 缓存文件路径
# levels 设置缓存文件目录层次；levels=1:2  表示两级目录
# keys_zone 设置缓存名字和共享内存大小
# inactive 在指定时间内没人访问则被删除
# max_size 最大缓存空间，如果缓存空间满，默认覆盖掉缓存时间最长的资源。
# 当配置好之后，重启nginx，如果不报错，则配置的proxy_cache会生效
# 查看proxy_cache_path /data/nginx/目录，会发现生成了tmp-test文件夹。

    location /tmp-test/ {  
	proxy_cache tmp-test; # 使用名为tmp-test的对应缓存配置
  	proxy_cache_valid  200 206 304 301 302 10d; # 对httpcode为200…的缓存10天 
	proxy_cache_key $uri; # 定义缓存唯一key,通过唯一key来进行hash存取
#	add_header X-Cache "$upstream_cache_status from $server_ip" # 可以添加一个首部，在客户端可以看见缓存是否命中
	proxy_set_header Host $host:$server_port; # 自定义http header头，用于发送给后端真实服务器。 
	proxy_set_header X-Real-IP $remote_addr;   
	proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;  
	proxy_pass http://127.0.0.1:8081/media_store.php/tmp-test/; # 指代理后转发的路径，注意是否需要最后的/

}

```
