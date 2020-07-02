# 配置 Let’s Encrypt SSL https 证书




> 配置 Let’s Encrypt SSL https 证书

# 一、安装acme.sh

```
curl  https://get.acme.sh | sh


#安装成功以后目录如下

/root/.acme.sh/

```



# 二、通过验证DNS签发证书

```
# GoDaddy 的域名

https://developer.godaddy.com/keys/


# 配置 两个变量
export GD_Key="sdfsdfsdfljlbjkljlkjsdfoiwje"
export GD_Secret="asdfsdafdsfdsfdsfdsfdsafd"

```


# 三、生成DNS TXT记录

```
acme.sh --issue --dns dns_gd -d jicki.cn


# 如果报错 可使用

acme.sh  --issue  --dns -d jicki.cn


# 会提示，手动创建一条 txt 记录

Add the following TXT record:
Domain: '_acme-challenge.jicki.cn'
TXT value: 'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
Please be aware that you prepend _acme-challenge. before your domain
so the resulting subdomain will be: _acme-challenge.jicki.cn
Please add the TXT records to the domains, and retry again.



# 等待解析完成之后, 重新生成证书:

acme.sh  --renew  -d jicki.cn

Renew: 'jicki.cn'
Single domain='jicki.cn'
Getting domain auth token for each domain
Verifying:jicki.cn
Success
Verify finished, start to sign.
Cert success.



# 如果解析未完成会提示

jicki.cn:Verify error:DNS problem: query timed out looking up CAA for jicki.cn

```


# 四、配置证书

```
acme.sh  --installcert  -d  jicki.cn   \
        --keypath   /etc/nginx/ssl/jicki.cn.key \
        --fullchainpath /etc/nginx/ssl/jicki.cn.cer
		
		
# 提示
Installing key to:/etc/nginx/ssl/jicki.cn.key
Installing full chain to:/etc/nginx/ssl/jicki.cn.cer
```


# 配置nginx

```
# 修改 nginx.conf 配置 增加 ssl

server {
    listen 80;
    listen [::]:80;
    server_name jicki.cn;

    # Redirect all HTTP requests to HTTPS with a 301 Moved Permanently response.
    return 301 https://$host$request_uri;
}

server
{
  listen     443;
  server_name www.jicki.cn jicki.cn;


  if ($http_user_agent ~ ApacheBench|WebBench|Jmeter){
       return 403;
     }

### Begin of SSL config
                ssl on;
                ssl_certificate /etc/nginx/ssl/jicki.cn.cer;
                ssl_certificate_key /etc/nginx/ssl/jicki.cn.key;
                ssl_session_timeout 1d;
                ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
                ssl_prefer_server_ciphers on;
                ssl_ciphers                EECDH+CHACHA20:EECDH+CHACHA20-draft:EECDH+AES128:RSA+AES128:EECDH+AES256:RSA+AES256:EECDH+3DES:RSA+3DES:!MD5;
                ssl_session_cache shared:SSL:50m;
                ssl_session_tickets      on;
                ssl_stapling      on;
                ssl_stapling_verify      on;
                resolver                 114.114.114.114 valid=300s;
                resolver_timeout         10s;

### End of SSL config
  location / {
     proxy_buffer_size 64k;
     proxy_buffers   32 32k;
     proxy_busy_buffers_size 256k;
     proxy_pass http://jekyll;
     proxy_cache one;
     }
 }

```


# FAQ:

## 官方文档

```
https://github.com/Neilpang/acme.sh/wiki/%E8%AF%B4%E6%98%8E
```

## 关于证书自动更新

```
官方提示
目前证书在 60 天以后会自动更新, 你无需任何操作. 今后有可能会缩短这个时间, 不过都是自动的, 你不用关心.
```


