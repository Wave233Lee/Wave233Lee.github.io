---
date: '2024-03-11T18:04:24+08:00'
title: 'MinIO 安装运行'
summary: "对象存储 MinIO 简单安装配置与运行"
tags: ["OSS"]
---

#### 使用Docker 运行单点模式
```bash
docker run -p 9000:9000 -di \
  --name minio \
  -v /data/minio:/data \
  -e "MINIO_ACCESS_KEY=HI3UNHSHXG2A0IR35YZO" \
  -e "MINIO_SECRET_KEY=bzXKoRrrqbpQ+1ezmJkBKxusVc5JZc02cBNm5vEd" \
  minio/minio server /data
```
| Parameter        | Description                              |
|------------------|------------------------------------------|
| -p               | 端口映射，将外部端口映射到容器内部端口                      |
| --name           | 自定义容器名称                                  |
| -di              | 后台运行的方式运行                                |
| --restart=always | 一旦docker重启或者开启时，也自动启动镜像                  |
| -e               | 设置系统变量，在这里是设置Minio的ACCESS_KEY和SECRET_KEY |
| -v               | 挂载文件，将系统文件映射到容器内部对应的文件夹                  |

#### 以普通用户身份运行MinIO Docker
Docker提供了标准化的机制，可以以非root用户身份运行docker容器。
```bash
mkdir -p ${HOME}/data
docker run -p 9000:9000 \
  --user $(id -u):$(id -g) \
  --name minio1 \
  -e "MINIO_ACCESS_KEY=AKIAIOSFODNN7EXAMPLE" \
  -e "MINIO_SECRET_KEY=wJalrXUtnFEMIK7MDENGbPxRfiCYEXAMPLEKEY" \
  -v ${HOME}/data:/data \
  minio/minio server /data
```
**使用二进制文件运行**
```bash
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
./minio server /data
```
#### MinIO客户端

- 下载二进制文件
```bash
wget -P /bin https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x /bin/mc
mc --help
```

- 添加一个云存储服务，也可直接修改~/.mc/config.json
```bash
# API签名为可选项 默认S3v4
mc alias set <ALIAS> <YOUR-S3-ENDPOINT> <YOUR-ACCESS-KEY> <YOUR-SECRET-KEY> --api <API-SIGNATURE> --path <BUCKET-LOOKUP-TYPE>
# 示例 MinIO本地存储
mc alias set minio http://localhost:9000 HI3UNHSHXG2A0IR35YZO bzXKoRrrqbpQ+1ezmJkBKxusVc5JZc02cBNm5vEd
# 示例 MinIO云存储
mc alias set minio http://192.168.1.51 BKIKJAA5BMMU2RHO6IBB V7f1CwQqAcwo80UEIJEjc5gVQUSSx5ohQ9GSrr12
# 示例 Amazon S3云存储
mc alias set s3 https://s3.amazonaws.com BKIKJAA5BMMU2RHO6IBB V7f1CwQqAcwo80UEIJEjc5gVQUSSx5ohQ9GSrr12
# 示例 Google云存储
mc alias set gcs  https://storage.googleapis.com BKIKJAA5BMMU2RHO6IBB V8f1CwQqAcwo80UEIJEjc5gVQUSSx5ohQ9GSrr12
```

- [MinIO Admin命令指南](https://docs.min.io/docs/minio-admin-complete-guide.html)

[官方文档](https://docs.min.io/docs/minio-docker-quickstart-guide.html)

