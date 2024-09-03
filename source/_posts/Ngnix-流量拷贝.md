---
title: Ngnix 流量拷贝
date: 2019-11-29 12:01:52
updated: 2019-11-29 12:01:52
categories: tech
tags: [nginx]
---

在需要真实的流量做旁路测试的时候，我们就可以使用 nginx 来做流量的拷贝
<!-- more -->
## 1 Nginx Mirror 模块
nginx 自1.13.4 版本开始，自带一个 [ngx_http_mirror_module](http://nginx.org/en/docs/http/ngx_http_mirror_module.html)，使用起来非常简单，只要在需要 mirror 的 location 处添加 mirror 关键字就行
```
location / {
    mirror /mirror;
    proxy_pass http://backend;
}
 
location = /mirror {
    internal;
    proxy_pass http://test_backend$request_uri;
}
```
一个简单的 nginx 配置如下：
``` 
http {
 
    upstream webserver1 {
        server  127.0.0.1:8881;
    }
 
    upstream webserver2 {
        server  127.0.0.1:8882;
    }
 
    server {
        listen       8080;
        server_name  localhost;
 
        location / {
            root   html;
            index  index.html index.htm;
        }
 
 
    location /test {
            mirror /mirror;
            proxy_pass http://webserver1/index;
        }
 
        location /mirror {
            internal;
            proxy_pass http://webserver2/index;
        }
 
    }
 
}
```

## 2 lua 脚本分发请求
但是 nginx 版本过低时，该如何做呢？
我们可以使用集成来 lua 的 [OpenResty](https://openresty.org/cn/)，使用 lua 脚本处理 web 请求完成流量的拷贝
我们直接看例子：
nginx.conf
```
user root;
worker_processes  1;
error_log logs/error.log;
events {
    worker_connections 1024;
}
http {
 
    upstream product {
            server  127.0.0.1:8881;
    }
    upstream test {
            server  127.0.0.1:8882;
    }
    server {
            listen      8080;
            server_name 172.27.133.28;
            #lua_code_cache off;
 
            # 外部应用的请求地址，通过 lua 脚本分发到新的 product 和 test 地址
            location /predict/fake {
                    client_body_buffer_size 2m;
                    set $switch    "on";              #开启或关闭copy功能
                    content_by_lua_file /tmp/copy_req.lua;
            }
             
            # lua 脚本分发到 product，并且对 url 重写后，反向代理到 product 的 server
            location ~* ^/product {
                    log_subrequest on;
                    rewrite ^/product(.*)$ /predict break;
                    proxy_set_header Host $host;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_pass http://product;
                    access_log logs/product-upstream.log;
            }
 
            # lua 脚本分发到 test，并且对 url 重写后，反向代理到 test 的 server
            location ~* ^/test {
                    log_subrequest on;
                    rewrite ^/test(.*)$ /predict break;
                    proxy_set_header Host $host;
                    proxy_set_header X-Real-IP $remote_addr;
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_pass http://test;
                    access_log logs/test-upstream.log;
            }
 
 
    }
 
}
```
copy_req.lua
```
local res1, res2, action
action = ngx.var.request_method
if action == "POST" then
        arry = {method = ngx.HTTP_POST, body = ngx.req.read_body()}
else
        arry = {method = ngx.HTTP_GET}
end
 
if ngx.var.switch == "on" then
        res1, res2 = ngx.location.capture_multi {    -- 分发请求的核心函数
                { "/product" .. ngx.var.request_uri , arry},
                { "/test" .. ngx.var.request_uri , arry},
        }
else
        res1, res2 = ngx.location.capture_multi {
                { "/product" .. ngx.var.request_uri , arry},
        }
end
 
if res1.status == ngx.HTTP_OK then
        local header_list = {"Content-Length", "Content-Type", "Content-Encoding", "Accept-Ranges"}
        for _, i in ipairs(header_list) do
                if res1.header[i] then
                        ngx.header[i] = res1.header[i]
                end
        end
        ngx.say(res1.body)
else
        ngx.status = ngx.HTTP_NOT_FOUND
end
```
我们来捋一捋这个流程：
背景：首先我们向外部注册的地址的是 /predict/fake，我们实际 web server 的地址是 /predict，然后一个生产的 upstream （product）一个测试用的 upstream（test），ok，这时候一个 /predict/fake 请求过来了发生了什么呢

1. 首先 /predict/fake 匹配到了 location，然后经过 lua 脚本分发请求，产生了两个子请求 /product/predict/fake 和 /test/predict/fake
2. /product/predict/fake 匹配到新的 location，并且对这个 url 进行了 rewrite，变成了真实的后端请求地址 /predict，并且反向代理到 product 这个 upstream 下的 server，同理对  /test/predict/fake 也是一样的
3. 两个子请求都有返回，但是在 lua 脚本中，只有 product 对应的请求被最终返回给了前端

需要注意的点：
反向代理时 rewrite 后的 url 会覆盖 proxy_pass 的 url

## 参考：
[nginx 之 proxy_pass 接口转发的规则](https://segmentfault.com/a/1190000018933857)
