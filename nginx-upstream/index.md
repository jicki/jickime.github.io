# Nginx 负载均衡模式



目前nginx负载均衡支持的5种方式的分配


## 轮询

每个请求按时间顺序逐一分配到不同的后端服务器,如果后端服务器down掉,能自动剔除.

```
upstream backserver {
        server 192.168.5.205;
        server 192.168.5.206;
}
```
 

## 权重(weight)

指定轮询几率,weight和访问比率成正比,用于后端服务器性能不均的情况.

```
upstream backserver {
        server 192.168.5.205 weight=10;
        server 192.168.5.206 weight=10;
}
```
 
## IP_hash

每个请求按访问ip的hash结果分配,这样每个访客固定访问一个后端服务器,可以解决session的问题.

```
upstream backserver {
        ip_hash;
        server 192.168.5.205:88;
        server 192.168.5.206:80;
}
```
 

## fair (第三方插件)

按后端服务器的响应时间来分配请求,响应时间短的优先分配.

```
upstream backserver {
        fair;
        server 192.168.5.205;
        server 192.168.5.206;
}
```
 

## url_hash(第三方)

按访问url的hash结果来分配请求,使每个url定向到同一个后端服务器,后端服务器为缓存时比较有效.

```
upstream backserver {
        server 192.168.5.205:3128;
        server 192.168.5.206:3128;
        hash $request_uri;
        hash_method crc32;
}
```
 


## 混合策略
Nginx 也支持多种策略混合使用

```
upstream backserver{
        ip_hash;
        server 127.0.0.1:9090 down; //(down 表示单前的server暂时不参与负载)
        server 127.0.0.1:8080 weight=2; //(weight 默认为1.weight越大,负载的权重就越大)
        server 127.0.0.1:6060;
        server 127.0.0.1:7070 backup; //(其它所有的非backup机器down或者忙的时候,请求backup机器)
        max_fails; //允许请求失败的次数默认为1.当超过最大次数时,返回proxy_next_upstream模块定义的错误
}
```

 
## 负载均衡健康检测

[健康检测模块][1]

需要打补丁，并且重新编译
解压 并 打上补丁...

```
nginx version >= 1.2.1
patch -p1 < check_1.2.1+.patch
nginx version < 1.2.1
patch -p1 < check.patch 
```

重新编译

```
./configure --user=www --group=www --prefix=/opt/local/nginx --with-http_stub_status_module  --with-http_ssl_module --add-module=/opt/software/yaoweibin-nginx_upstream_check_module-dfee401/
make && make install
```
 

修改配置文件：

```
upstream webserver {
        server 10.3.0.100:8000 weight=1;
        server 10.3.0.101 weight=2;
        check interval=1000 rise=2 fall=2 timeout=1000; 
        }
```

interval检测间隔时间，rsie请求2次正常的话为up,fail请求2次失败的话为down,timeout检查超时时间(毫秒)
check_http_send "GET /.test.html HTTP/1.0";  //所发送的检测请求 


``` 
server {  
        listen       80;  
        server_name  localhost;  
        location / {  
        proxy_pass http://peace;  //引用  
        }     
location /status {   //定义一个类似stub_status的方式输出检测信息  
    check_status;  
 }  
        error_page   500 502 503 504  /50x.html;  
        location = /50x.html {  
            root   html;  
        }  
    }  
} 
```

  [1]: https://github.com/yaoweibin/nginx_upstream_check_module/ "模块下载"
  

