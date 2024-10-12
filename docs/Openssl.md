# 通过 Openssl 工具生成自签名证书

## 简介

在部署 Web 网站的时候，需要申请 SSL/TLS 证书才能开启 Https 访问，而免费的证书申请机构 Let’s Encrypt 所颁发的证书一般只有三个月有效期，虽然可以通过脚本实现自动续期，但如果只是个人使用又或者是进行测试的话，使用可以自定义有效期的自签名证书会更方便一点。本文将使用开源工具 Openssl 来实现自签名证书的生成。

## Openssl 下载安装

官方仓库并没有发布可执行文件，不过在官网维护了一个第三方下载的地址列表：https://wiki.openssl.org/index.php/Binaries

我是在 https://slproweb.com/products/Win32OpenSSL.html 这个网站下载的，其中有 `Win64 OpenSSL v3.3.2 Light` 和 `Win64 OpenSSL v3.3.2` 两种版本，含 Light 的是轻量版本，安装后只有 17MB 左右，完整版需要 700MB 左右，这里选择安装轻量版就够用了。

安装后程序的可执行文件位于程序根目录中的 `bin` 目录，如果需要全局命令行访问需要把 `OpenSSL-Win64-light\bin` 添加到环境变量的 `PATH` 中（注：该文章所有操作均在 Win 系统下完成，Linux 系统自行融会贯通）。

## 证书颁发流程

?> 注：此小节为原理，如果你只想快速生成证书请看 [实操](#实际操作) 篇

### 1. 生成私钥

生成私钥有以下两个不同的参数可选：

- genpkey
- genrsa

以下分别展示两种命令的用法：

#### genpkey

```bash
openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out private.pem
```

`-algorithm RSA`：使用 RSA 算法。

`-pkeyopt rsa_keygen_bits:2048`：RSA 密钥长度为 2048。

> RSA 加密算法：密钥位数越长，越安全，但速度越慢，一般推荐 2048 位，目前广泛使用，其他加密算法如：ECDSA、EdDSA、DSA 等自行了解。

以上命令执行后会生成私钥文件 `private.pem` ，

#### genrsa

> 该命令专门用于生成 `RSA` 私钥。

```bash
# 生成一个 4096 位的 RSA 私钥
openssl genrsa -out private.pem 4096

# 查看私钥文件信息
 openssl rsa -in private.pem -text -noout -check
```

### 2. 生成证书请求 csr 文件

```bash
# 生成 request.csr 文件
openssl req -new \
    -key private.pem \
    -out request.csr \
    -subj "/C=CN/ST=Zhejiang/L=Hangzhou/O=Example Inc/OU=IT/CN=acbye.cn/emailAddress=ac@acbye.cn"

# 查看 csr 文件信息
openssl req -in request.csr -text -noout
```

`-key private.key`：指定用于生成 CSR 的私钥文件。

`-subj`：指定主体信息，以 `/` 开头，后面跟着一系列键值对，格式为 `key=value`。这些键值对包括：

- C：国家代码（两位字母），如：CN
- ST：省份或州名，如：Zhe jiang
- L：城市名，如：Hang zhou
- O：组织名，如：Example Inc
- OU：组织部门名，如：IT
- CN：通用名称（通常是域名或服务器名），如：acbye.cn
- emailAddress：电子邮件地址，如：ac@acbye.cn

还可以一条命令同时生成私钥和 CSR 文件：

```bash
openssl req -new \
    -newkey rsa:2048 \
    -keyout private.pem \
    -nodes \
    -out request.csr \
    -subj "/C=CN/ST=Zhejiang/L=Hangzhou/O=Example Inc/OU=IT/CN=acbye.cn/emailAddress=ac@acbye.cn"
```

`-nodes`：不使用密码

### 3. 提交 CSR 文件给 CA 机构申请证书

这一步需要提交生成的 CSR 文件给 CA 机构，在通过验证后就可以得到颁发的证书文件，不过由于我们是要创建自签名证书，所以我们自己就是一个 CA，创建证书的命令如下：

```bash
openssl x509 -req \
    -days 3650 \
    -in request.csr \
    -sha256 \
    -signkey private.pem \
    -out cert.crt

# 查看证书信息
openssl x509 -in cert.crt -text -noout
```

`x509`：生成 X509 标准的证书。

`-days 3650`：证书有效期 3650 天。

`-in request.csr`：证书申请文件。

`sha256`：使用 SHA256 算法进行签名。

`-signkey private.pem`：指定私钥文件，用于对生成的证书进行签名，用自己的私钥给自己的证书签名就是自签名证书。

经过上述步骤就得到了可以使用的自签名证书，不过这套流程下来未免太过于繁琐，接下来是 Show Time 😏。

## 实际操作

```bash
openssl req -x509 \
    -newkey rsa:4096 \
    -sha256 \
    -days 3650 \
    -nodes -keyout rootCA-key.pem \
    -subj "/C=CN/ST=Fu Jian/L=FuZhou/O=IT/CN=AC Personal CA/emailAddress=ca@acbye.cn" -out rootCA.crt
```

只需上面这一行命令，就可以生成自签名证书了，岂不妙哉 😏。

不过自签名证书是不被浏览器信任的，需要把证书导入到系统 **_受信任的证书颁发机构_** 中解决，如果你需要多个自签名证书，这样一个个导入挺麻烦的，所有我们可以生成一个 10 年的根证书，然后用这个根证书去签名其他的证书，这样就只需要把根证书导入系统即可。

上面的命令示例其实就是生成了一个根证书，他与普通证书无异，只是名字叫这个罢了，下面演示怎么用它去签名别的证书：

```bsah
openssl req -x509 \
    -newkey rsa:2048 \
    -sha256 \
    -days 900 \
    -nodes -keyout acbye.cn-key.pem \
    -CA rootCA.crt \
    -CAkey rootCA-key.pem \
    -subj "/C=CN/ST=Fu Jian/L=FuZhou/O=IT/CN=acbye.cn/emailAddress=ac@acbye.cn" \
    -addext "subjectAltName=DNS:acbye.cn,DNS:*.acbye.cn" -out acbye.cn.crt
```

`-CA`：即为要使用的根证书。

`-CAkey`：为根证书的私钥。

`subjectAltName=DNS:acbye.cn,DNS:*.acbye.cn`：添加域名，多个域名用逗号隔开，其中 `*.acbye.cn` 为通配符，表示 acbye.cn 及其子域名都可以用这个证书，又称为泛域名证书。

`-out acbye.cn.crt`：证书文件。

`-keyout acbye.cn-key.pem`：证书的私钥文件。

现在你已经学会了创建自签名证书了，接下来快去手搓一个网站吧 😆。
