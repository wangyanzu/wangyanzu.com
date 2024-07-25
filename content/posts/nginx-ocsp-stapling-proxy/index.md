---
title: "使用OCSP Stapling Proxy加速网站响应"
date: 2024-07-24T22:30:00+08:00
# weight: 1
# aliases: ["/first"]
tags:
- ocsp stapling
- golang
categories:
- nginx
author: "WANG"
# author: ["Me", "You"] # multiple authors
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "使用OCSP Stapling Proxy加速网站响应"
canonicalURL: "https://canonical.url/to/page"
disableHLJS: true # to disable highlightjs
disableShare: false
disableHLJS: false
hideSummary: false
searchHidden: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
cover:
    image: "posts/nginx-ocsp-stapling-proxy/nginx.gif" # image path/url
    alt: "redis-logo" # alt text
    caption: "<text>" # display caption under cover
    relative: true # when using page bundles set this to true
    hidden: false # only hide on current single page
# editPost:
#     URL: "https://github.com/<path_to_repo>/content"
#     Text: "Suggest Changes" # edit text
#     appendFilePath: true # to append file path to Edit link
---
## OCSP Stapling
### 什么是 OCSP Stapling
OCSP装订（英语：OCSP Stapling），正式名称为TLS证书状态查询扩展，可代替在线证书状态协议（OCSP）来查询X.509证书的状态。服务器在TLS握手时发送事先缓存的OCSP响应，用户只需验证该响应的有效性而不用再向数字证书认证机构（CA）发送请求。
### 存在问题
如上所述，当用户请求网站时，需要进行OCSP查询来确定证书状态。然而 let's encrypt 服务器并不在国内，访问速度慢，且域名记录容易被污染，就有可能造成
- 访问速度慢造成打开网页卡顿一段时间后才能顺利访问网站
- 域名、ip被污染或者墙，则无法打开网站
### 如何解决
使用OCSP装订在服务端直接查询证书状态，并将结果一并返回给客户端，用户不再需要主动访问ocsp服务器检查证书。
## 简单开启 OCSP Stapling
### 增加NGINX装载配置
```
ssl_stapling on;
resolver 223.5.5.5 114.114.114.114 valid=300s;
resolver_timeout 5s;
```
在`http`或者`server`增加以上配置，则已经开启了OCSP装订功能。
### 检查装订效果
#### 检查命令
```bash
HOST="www.example.com"
openssl s_client -connect ${HOST}:443 -servername ${HOST} -status -tlsextdebug < /dev/null 2>&1 | grep -i "OCSP response"
```
#### 检查结果
> 注意，您可能需要多次请求才能得到已装订的结果，这取决于nginx的woker数量
##### 未装订
```
OCSP response: no response sent
```
##### 已经装订
```
OCSP response:
OCSP Response Data:
    OCSP Response Status: successful (0x0)
    Response Type: Basic OCSP Response
```
## 使用OCSP代理
### 还存在的问题
通常，简单开始nginx OCSP Staping 功能已经满足大多数用户的需求，但是还可能存在以下问题：
- 网站第一次访问或者缓存过期，仍然返回的不是已装订的响应
- 在OCSP服务器受到污染的情况下，缓存过期后就无法返回已装订的响应
- NGINX所在服务器无法直接访问公网，无法获取到OCSP响应
### 使用代理
为了解决以上问题，发现了一个 https://github.com/dlecorfec/ocsp-proxy 项目使用简单的代码完成了ocsp的代理。

为了确保ocsp装订的成功率，对项目做了简单的修改，增加了缓存功能。

项目地址为 https://github.com/wangyanzu/ocsp-proxy

#### 获取证书OCSP服务器
```bash
openssl x509 -noout -ocsp_uri -in example.com.crt
```
#### 启动服务器 docker 方式
> 指定真实的OCSP服务器，并启动容器

docker run -d -p8080:8080 -e OCSP_HOST:"e5.o.lencr.org" iwangs/ocsp-proxy 

#### 调整NGINX配置
```
ssl_stapling on;
ssl_stapling_responder http://10.2.2.2:8080/;
resolver 223.5.5.5 114.114.114.114 valid=300s;
resolver_timeout 5s;
```
在`http`或者`server`增加以上配置，请求自己的OCSP代理服务器。

#### 最后
- 通常OCSP响应有效期为`7`天，即7天内，使用缓存装订都没有问题
- nginx自身ocsp缓存为`1`小时（有效期对此时间有影响）
- 装订失败会回到不装订的响应，不会直接影响用户的请求响应