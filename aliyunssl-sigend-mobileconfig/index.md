# Aliyun SSL 签注 mobileconfig


# 签注 mobileconfig

## 申请证书

* 登录阿里云SSL证书申请


https://www.aliyun.com/product/cas?utm_medium=text&utm_source=baidu&utm_campaign=anquan&utm_content=se_1000125802


* 选择免费版个人DV证书 --> Symantec --> 购买


## 验证证书

* 购买成功, 转到控制台 --> 证书申请 --> 填写基本信息 (域名相关) --> 验证服务器(DNS 与 文件)

## 下载证书

1. 下载证书 --> 需要下载 Apache 与 Nginx 这两个证书文件,两个都要下载

2. 需要用到 Apache 证书的 xxx_xxx.com_public.crt 文件

3. 需要用到 Nginx 证书的 xxx_xxx.com.key 与 xxx_xxx.com.pem 文件

## 修改证书

1. 将 public.crt 重命名为 mbaike.crt   (https服务器端使用公钥证书文件） 

2. 将 key  重命名为 mbaike.key     （https服务器端使用证书对应的key)

3. 将 pem  重命名为 ca-bundle.pem    (https服务器端使用证书pem)

4. 需要 unsigned.mobilecofig 文件   （IOS端生成的未签名的配置描述文件）

## 签注 文件

```shell
openssl smime -sign -in unsigned.mobileconfig -out signed.mobileconfig -signer mbaike.crt -inkey mbaike.key -certfile ca-bundle.pem -outform der -nodetach
```

* 生成 signed.mobileconfig 文件


