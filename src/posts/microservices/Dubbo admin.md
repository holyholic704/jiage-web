---
date: 2024-08-01
category:
  - 微服务
tag:
  - Dubbo
  - 环境配置
excerpt: Docker 部署 Dubbo admin
order: 99
---

# 部署 Dubbo admin

项目包括前端模块（dubbo-admin-ui），且该模块经常会打包失败，推荐使用 Docker 部署

拉取最新的镜像

```shell
docker pull apache/dubbo-admin:latest
```

编写配置文件，或去 [application.properties](https://github.com/apache/dubbo-admin/blob/develop/dubbo-admin-server/src/main/resources/application.properties) 下载

- 推荐使用 `-v` 挂载配置文件
- 不推荐使用 `-e` 设置启动参数，如果 Nacos 配置了账号密码访问等需要拼接字符串，要进行转义

```properties
# 集群配置一个节点即可
admin.registry.address=nacos://XXX.XXX.XXX.XXX:8848?username=Nacos账号&password=Nacos密码&group=dev_group&namespace=68a0e6f6-210a-4d12-85ff-533d6daa9264
admin.config-center=nacos://XXX.XXX.XXX.XXX:8848?username=Nacos账号&password=Nacos密码&group=dev_group&namespace=68a0e6f6-210a-4d12-85ff-533d6daa9264
admin.metadata-report.address=nacos://XXX.XXX.XXX.XXX:8848?username=Nacos账号&password=Nacos密码&group=dev_group&namespace=68a0e6f6-210a-4d12-85ff-533d6daa9264
admin.root.user.name=登录 Dubbo admin 的账号
admin.root.user.password=登录 Dubbo admin 的密码
```

启动

```shell
docker run -d -p 38080:38080 --name dubbo-admin -v 本地存放application.properties的目录:/config apache/dubbo-admin
```

## 参考

- [dubbo-admin
](https://github.com/apache/dubbo-admin)
