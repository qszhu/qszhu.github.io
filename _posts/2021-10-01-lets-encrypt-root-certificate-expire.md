---
layout: post
title: 国庆生产事故！
date: 2021-10-01 20:00:00 +0800
tags: [lets_encrypt, certificate, expire, dev_ops]
---

Let's Encrypt小半年前发了个公告[1]，说它的根证书9月30号就要过期了。当时我评估了一下，觉得应该没啥影响。不料国庆一大早就收到用户报告说有页面缺失。一路排查，发现调用链中有个http接口返回了证书过期错误。然而同样的接口如果用浏览器访问的话是没问题的。于是问题应该是发起请求的那台服务器的根证书没有更新。

由于是个内部接口，所以决定先暂时忽略错误再慢慢修。因为是个node.js服务，通过设置环境变量`NODE_TLS_REJECT_UNAUTHORIZED=0`就可以。不过环境变量会影响当前进程及其子进程，影响范围比较大，就只改了出错的接口：

```javascript
import https from 'https'

const request = axios.create({
  httpsAgent: new https.Agent({
    rejectUnauthorized: false
  })
});

// ...
return request(config)
```

接口恢复以后开始检查环境。因为是台旧服务器，openssl即便更新后版本也比较低：

```bash
$ sudo apt install openssl libgnutls-openssl27 libgnutls30
$ openssl version
OpenSSL 1.0.2g  1 Mar 2016
```

从公告上看，就是受影响的版本，需要升级根证书：

```bash
$ sudo apt install ca-certificates
```

升级后发现用`curl`可以访问了，不过`wget`还不行。网上搜了圈说试试手动去掉过期的证书：

```bash
$ sudo sed -i 's/mozilla\/DST_Root_CA_X3.crt/!mozilla\/DST_Root_CA_X3.crt/g' /etc/ca-certificates.conf
$ sudo update-ca-certificates
```

这下`curl`和`wget`都可以了，不过node.js发起的请求还是会返回证书过期。原地裂开...

看了下node.js的文档[2]，node的每个版本都硬编码了根证书。解决办法文档上也给出了，就是启动时使用`--use-openssl-ca`命令行参数：

```bash
$ node ---use-openssl-ca start.js
```

`pm2`的话直接更新启动参数：

```bash
$ pm2 restart xxxService --node-args "--use-openssl-ca"
```

测试可行。不过另一个有问题的服务在docker镜像里，就算让node.js使用openssl的根证书，那也是base镜像里的。要修改的话就要更新base镜像并重新生成了。如果无法升级base镜像或者需要重新打包的镜像很多的话就很麻烦。

好在还可以直接挂载目录：

```yaml
command: node --use-openssl-ca start.js
volumes:
  - /etc/ssl:/etc/ssl:ro
env_file: .env
```

再设置node.js环境变量`SSL_CERT_FILE`就好了：

```
SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
```

总结一下修复步骤：

0. 应急时可设置`NODE_TLS_REJECT_UNAUTHROIZED=0`，修复后去掉
1. 升级`openssl`和根证书
2. 去掉过期的根证书
3. 挂载系统根证书目录到镜像
4. 设置`SSL_CERT_FILE`路径，启动时添加`--use-openssl-ca`参数

外网搜了一圈，似乎国外不少运维都度过了一个难忘的夜晚，国庆长假第一天加班的我心里稍微平衡了一点...

## 参考资料

* [1] [DST Root CA X3 Expiration (September 2021) -  Let’s Encrypt](https://letsencrypt.org/docs/dst-root-ca-x3-expiration-september-2021/)
* [2] [Command-line options \| Node.js v16.10.0 Documentation](https://nodejs.org/api/cli.html#cli_use_bundled_ca_use_openssl_ca)