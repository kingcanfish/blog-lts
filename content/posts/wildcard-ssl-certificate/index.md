---
title: " 使用acme.sh配置https的证书"
date: 2019-09-10T14:35:00+08:00
tag: [https, nginx, ssl]
description: "使用acme.sh配置https的证书"
---
### 写在前面吼

**由于有了`acme.sh` 以及`Let's Encrypt` 公司（或组织）的无私贡献**
**也让我们这种小玩家能够用上https，并且还是泛域名的解析**
**一来保证了数据的传输安全，二来提升了逼格（直至现在大部分NCU的域名也都没有上HTTPS）**
**在此感谢acme.sh 以及 Let's Encrypt 的无私贡献**
___


### 1.首先当然是安装 acme.sh
>  curl https://get.acme.sh | sh

可选择在.bashrc或者是如果zsh使用者可以使用.zshrc添加映射关系：
>alias acme.sh=~/.acme.sh/acme.sh

### 2.获取dns_api
我用的是阿里云的ECS，所一这里只拿阿里云的DNS举例子，其他的DNS可以用过以下链接进行查找
>[dnsapi · Neilpang/acme.sh Wiki · GitHub](https://github.com/Neilpang/acme.sh/wiki/dnsapi)
***

通过地址`[阿里云dns](https://ak-console.aliyun.com/#/accesskey)`获得你的阿里云账号的
`Ali_Key` 和 `Ali_Secret`

然后在环境变量中导入

> export Ali_Key="\*\*\*\*\*\*\*\*\*\*\*" 
> export Ali_Secret="\*\*\*\*\*\*\*\*\*\*\*"


### 3.安装证书

> acme.sh --issue -d \*\*\*\*\*\* --dns dns_ali

就会在 `~/.acme.sh`目录下生成证书秘钥，然后再copy 到制定存放证书的目录，然后配置下nginx的配置文件就ok了



---

2020.6.2 更新，附上一个自动更换证书脚本

```python
import os
from datetime import datetime

cert_path = '/etc/nginx/cert'

def log(*args, **kwargs):
    print("{0}:".format(datetime.now().strftime("%Y-%m-%d %H:%M:%S")), *args, **kwargs)


log("♥签发证书...")
log("♥让他飞一会")
r = os.system("/home/kuo/.acme.sh/acme.sh "
              "--issue --force -d guoxy.top -d *.guoxy.top  --dns dns_ali "
              "--key-file {c}/guoxy.top.key --fullchain-file {c}/fullchain.cer".format(c=cert_path))
if r != 0:
    exit(-1)

# 重启openresty
log("♥重启nginx...")
r = os.system("sudo nginx -s reload")
if r != 0:
    exit(-1)
log ("🆗 nginx重启成功~"))

log ("🆗 恭喜又白嫖了三个月~")
```





___


附上一片参考博客，致谢
[泛域名：使用acme.sh免费自动签发 Let's Encrypt 泛域名证书-vadxq-清竹茶馆](https://blog.vadxq.com/article/acmesh-letsencrypt)


