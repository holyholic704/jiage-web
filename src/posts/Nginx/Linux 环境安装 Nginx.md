# Linux 环境安装 Nginx

## 安装

```shell
# 下载，推荐使用最新的稳定版本
wget http://nginx.org/download/nginx-1.26.1.tar.gz

# 解压
tar -zxvf nginx-1.26.1.tar.gz

# 执行编译前的环境配置（默认安装路径为：/usr/local/nginx）
# --prefix 设置安装路径
# --with-http_stub_status_module --with-http_ssl_module 是 SSL 证书相关模块，没有需要可以不添加
./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module

# 编译及安装
make && make install

# 依赖（如果需要）
#   gcc编译工具
#   pcre库
#   zlib库
#   openssl库
yum install -y gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel

# 启动，进入/usr/local/nginx/sbin
./nginx

# 设置软链接，之后就无需再到目录中执行了
ln -s /usr/local/nginx/sbin/nginx /usr/bin/
```

- 执行成功后，默认会产生两个服务进程，一个 master 进程，一个 worker 进程
- 默认端口号为 80

## 卸载

```shell
# 关闭进程
./nginx -s stop

# 删除文件夹
rm -rf /usr/local/nginx

# 清除编译环境，进入到源代码解压后的目录
make clean
```

## 配置 SSL

首先查看 nginx 是否安装 http_ssl_module 模块

```bash
./nginx -V
```

![](./md.assets/20240531104534.png)

这一步可以省略，无论检不检查，没装的话你是用不了的 `^_^`

### 如果安装时没有添加 SSL 模块如何处理

进入到解压目录执行以下命令，注意不是安装目录

- 解压目录：下载 nginx 压缩包后解压的目录，也可叫做源码目录
- 安装目录：安装时如没特别指定的话，一般在 `/usr/local/nginx`

```shell
./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module
```

执行成功后进行编译 `make`

- 切记不要使用 `make install` 进行编译，会覆盖原来安装的文件

关闭并备份可执行文件，如没有备份的需要，可以省略以下备份步骤

```bash
nginx -s stop
```

```bash
cp /usr/local/nginx/sbin/nginx /usr/local/nginx/sbin/nginx.bak
```

编译完成后，会在解压目录中生成一个 objs 文件，将里面的 nginx 文件复制到安装目录中就 OK 了

```bash
cp objs/nginx /usr/local/nginx/sbin/
```

### SSL 配置

```bash
worker_processes  auto;

events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    server {
        listen       80;
        server_name  你的域名;

        location / {
            root   你的项目地址;
            index  index.html index.htm;
        }

        # 将请求转成 https
        rewrite ^(.*)$ https://$host$1 permanent;

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }

    # SSL配置
    server {
        listen       443 ssl;
        server_name  你的域名;

        # 配置下载下来的的证书文件
        ssl_certificate      /usr/local/nginx/cert/你的证书.crt;
        ssl_certificate_key  /usr/local/nginx/cert/你的证书.key;

        ssl_session_cache    shared:SSL:50m;
        ssl_session_timeout  1d;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers  ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
        ssl_prefer_server_ciphers  on;

        location / {
            root   你的项目地址;
            index  index.html index.htm;
        }
    }
}
```

配置完成，执行 `nginx -t` 检查下配置，不通过的话会提示你哪些地方有错误，修改之后启动 nginx 即可开始享受 HTTPS 了

## 参考

- [nginx: download](http://nginx.org/en/download.html)
- [Linux Nginx的安装与配置（全程图文记录超详细）](https://blog.csdn.net/qq_39420519/article/details/126322909)
- [Nginx配置Https（详细、完整）](https://www.cnblogs.com/ambition26/p/14077773.html)
- [Nginx 服务器 SSL 证书安装部署（Linux）](https://cloud.tencent.com/document/product/400/35244?from_cn_redirect=1)
- [nginx启动报 ssl parameter requires ngx_http_ssl_module](https://blog.csdn.net/qq_40390762/article/details/131879841)
- [nginx improving SSL Configuration](https://gist.github.com/mohanpedala/9f6bc9f9f6a50fd04c9d09dc95155fe3)
