<div align="center">

# docker-proxy

[![Auth](https://img.shields.io/badge/Auth-kubesre-ff69b4)](https://github.com/kubesre)
[![GitHub contributors](https://img.shields.io/github/contributors/kubesre/docker-registry-mirrors)](https://github.com/kubesre/docker-registry-mirrors/graphs/contributors)
[![GitHub Issues](https://img.shields.io/github/issues/kubesre/docker-registry-mirrors.svg)](https://github.com/kubesre/docker-registry-mirrors/issues)
[![GitHub Pull Requests](https://img.shields.io/github/issues-pr/kubesre/docker-registry-mirrors)](https://github.com/kubesre/docker-registry-mirrors/pulls)
[![GitHub Pull Requests](https://img.shields.io/github/stars/kubesre/docker-registry-mirrors)](https://github.com/kubesre/docker-registry-mirrors/stargazers)
[![HitCount](https://views.whatilearened.today/views/github/kubesre/docker-registry-mirrors.svg)](https://github.com/kubesre/docker-registry-mirrors)
[![GitHub license](https://img.shields.io/github/license/kubesre/docker-registry-mirrors)](https://github.com/kubesre/docker-registry-mirrors/blob/main/LICENSE)

<p> 自建多平台容器镜像代理服务,支持 Docker Hub, GitHub, Google, k8s, Quay, Microsoft 等镜像仓库. </p>

<img src="https://cdn.jsdelivr.net/gh/kubesre/tu@main/img/image_20240420_214408.gif" width="800"  height="3">
</div><br>

# 📝准备工作
⚠️ 重要：一台国外的服务器[腾讯云特惠服务器推荐](https://curl.qcloud.com/TJMyJvrd)，并且未被墙。一个域名，无需国内备案，便宜的就行(推荐xyz结尾的，首年最低7元)！通过脚本可自动实现HTTPS。


使用脚本前请确认域名的[@记录和*记录]已经解析到该服务器！


# 🚀快速开始

## 使用docker compose部署(自动配置https证书)


  
⚠️ 前提: 准备一个域名并做好 DNS 解析到准备好的服务器的 IP

1. 在服务器里新建一个文件 docker-compose.yaml 内容如下
```
version: '3'
services:
  crproxy:
    image: ghcr.io/daocloud/crproxy/crproxy:v0.9.1
    container_name: crproxy
    restart: unless-stopped
    ports:
    - 80:8080
    - 443:8080
    command: |
      --acme-cache-dir=/tmp/acme
      --acme-hosts=*
      --default-registry=docker.io
    tmpfs:
      - /tmp/acme
    
    # 非必须, 如果这台服务器无法畅通的达到你要的镜像仓库可以尝试配置 
    #environment:
    #- https_proxy=http://proxy:8080
    #- http_proxy=http://proxy:8080
```

2.然后执行 `docker-compose up -d`


3.然后就能愉快的拉取镜像了

``` shell
docker pull 你的域名/hello-world
```

4.也可以添加到 /etc/docker/daemon.json

``` json
{
  "registry-mirrors": [
    "https://你的域名"
  ]
}
```

``` shell
docker pull hello-world
```
<details>
<summary><strong>进阶版本docker-compose（支持别名）</strong></summary>
  
### 进阶版本docker-compose（支持别名）
1. 在服务器里新建一个文件 docker-compose.yaml 内容如下

```
version: '3'
services:
  crproxy:
    #image: ghcr.io/daocloud/crproxy/crproxy:v0.10.0
    image: ghcr.io/daocloud/crproxy/crproxy:v0.13.0-alpha.15-4
    container_name: crproxy
    restart: unless-stopped
    ports:
    - 80:8080
    - 443:8080
    command: |
       -a :8080
       --enable-pprof true
       #配置dockerhub认证用户，解决默认用户拉取限制
       #-u username:password@docker.io
       #配置存储，使用oss存储镜像
       #--storage-driver oss
       #--storage-parameters region=oss-ap-southeast-1,accesskeyid=LCskAz6CXEWb,accesskeysecret=8KSWMt1SXfbdoMFfk0BJuO2WKYGqGM,bucket=fanzhi-data
       #--storage-parameters region=oss-ap-southeast-1,accesskeyid=LTAI5tAcZLz6CXEWb,accesskeysecret=8KSWMt1SXfbdoMFfk0BJuO2WKYGqGM,bucket=docker-registry-mirrors,internal=true
       --retry 3
       --retry-interval 3s
       --disable-keep-alives nvcr.io
       --behind true
       --disable-tags-list
       --limit-delay
       --privileged-no-auth    
       --ips-speed-limit 1Gi/h
       --blobs-speed-limit 1024Ki/s
       --simple-auth
       --token-url "https://你的域名/auth/token"
       --acme-cache-dir=/tmp/acme
       --acme-hosts=*
       #配置默认仓库
       #--default-registry=docker.io
       #配置仓库别名
       --override-default-registry=docker.你的域名=docker.io
       --override-default-registry=l5d.你的域名=cr.l5d.io
       --override-default-registry=elastic.你的域名=docker.elastic.co
       --override-default-registry=gcr.你的域名=gcr.io
       --override-default-registry=ghcr.你的域名=ghcr.io
       --override-default-registry=k8s-gcr.你的域名=k8s.gcr.io
       --override-default-registry=k8s.你的域名=registry.k8s.io
       --override-default-registry=mcr.你的域名=mcr.microsoft.com
       --override-default-registry=nvcr.你的域名=nvcr.io
       --override-default-registry=quay.你的域名=quay.io
       --override-default-registry=jujucharms.你的域名=registry.jujucharms.com
       --simple-auth
       #添加认证用户
      # --simple-auth-user username=password
    tmpfs:
      - /tmp/acme
```
注意修改`你的域名` 为你自己的域名


2.然后执行 `docker-compose up -d`

3.使用方式参考下方别名方案

<div>
</details>
  
<del>
  
## 通过项目脚本部署

<details>
<summary><strong>通过项目脚本部署</strong></summary>
<div>
  
```
# CentOS
yum -y install wget curl
# ubuntu
apt -y install wget curl

bash -c "$(curl -fsSL https://raw.githubusercontent.com/kubesre/docker-registry-mirrors/main/dockerproxy/install/DockerProxy_Install.sh)"
```
</details>
</del>
<del>
  
## 使用Render部署（无需服务器和域名且免费方案）
<details>
<summary><strong>部署到 Render</strong></summary>
<div>

[使用Render快速部署](Render/README.md)

</details>

</del>

## 使用Sealos部署（无需服务器和域名-个人使用低成本方案）
<details>
<summary><strong>部署到 Sealos</strong></summary>
<div>

[使用Sealos快速部署](Sealos/README.md)

</details>

## 🔨 功能

- 一键部署Docker镜像代理服务的功能,并且自动配置https证书.
- 支持多个镜像仓库的代理，包括Docker Hub、GitHub Container Registry (ghcr.io)、Quay Container Registry (quay.io)和 Kubernetes Container Registry (k8s.gcr.io)
- 自动检查并安装所需的依赖软件，如Docker、Nginx等，并确保系统环境满足运行要求.
自动清理注册表上传目录中的那些不再被任何镜像或清单引用的文件
- 提供了重启服务、更新服务、更新配置和卸载服务的功能，方便用户进行日常管理和维护
- 支持主流Linux发行版操作系统,例如Centos、Ubuntu、Rocky、Debian、Rhel等
支持主流ARCH架构下部署，包括linux/amd64、linux/arm64
# ✨ 教程
## 前缀替换的 Registry 的参考
| 源站	                 | 替换为              |
|--------------------------|------------------------------|
| cr.l5d.io                | l5d.your_domain_name              |
| docker.elastic.co        | elastic.your_domain_name          |
| docker.io                | docker.your_domain_name           |
| gcr.io                   | gcr.your_domain_name              |
| ghcr.io                  | ghcr.your_domain_name             |
| k8s.gcr.io               | k8s-gcr.your_domain_name          |
| registry.k8s.io          | k8s.your_domain_name              |
| mcr.microsoft.com        | mcr.your_domain_name              |
| nvcr.io                  | nvcr.your_domain_name             |
| quay.io                  | quay.your_domain_name             |
| registry.jujucharms.com   | jujucharms.your_domain_name       |
## 使用方法
### 以argocd 清单文件为例：
```bash
wget https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 首先需要确定原始镜像地址仓库
以argocd yaml文件举例：
```bash
grep -n image: install.yaml
21645:        image: quay.io/argoproj/argocd:v2.11.0
21739:        image: ghcr.io/dexidp/dex:v2.38.0
21768:        image: quay.io/argoproj/argocd:v2.11.0
21850:        image: quay.io/argoproj/argocd:v2.11.0
21927:        image: redis:7.0.14-alpine
22162:        image: quay.io/argoproj/argocd:v2.11.0
22214:        image: quay.io/argoproj/argocd:v2.11.0
22531:        image: quay.io/argoproj/argocd:v2.11.0
22825:        image: quay.io/argoproj/argocd:v2.11.0
```
### 方案一
**使用方式：**

使用方式都是替换原来镜像的前缀域名即可实现加速效果，比如：
```bash
#docker.io
原来地址： redis:7.0.14-alpine  # 这个是官方镜像，省略了前边的域名
替换地址： docker.your_domain_name/redis:7.0.14-alpine
#quary.io
原来的地址： quay.io/argoproj/argocd:v2.11.0
替换地址： quay.your_domain_name/argoproj/argocd:v2.11.0
#ghcr.io
原来的地址： ghcr.io/dexidp/dex:v2.38.0
替换地址： ghcr.your_domain_name/dexidp/dex:v2.38.0
```
### 方案二
#### 注意事项
通过这种方式只能加速docker hub的镜像，对于其他镜像仓库，比如k8s.gcr.io, quay.io等，需要使用**方案一**替换前缀的方式进行加速。
#### 使用方式：
还有一种方案是通过将加速地址写入到docker配置文件当中实现加速。

**Ubuntu14.04、Debian7Wheezy**

对于使用 upstart 的系统而言，编辑 /etc/default/docker 文件，在其中的 DOCKER_OPTS 中配置加速器地址：
```Bash
DOCKER_OPTS="--registry-mirror=https://docker.your_domain_name"

```
**Ubuntu16.04+、Debian8+、CentOS7**


对于使用 systemd 的系统，请在 /etc/docker/daemon.json 中写入如下内容（如果文件不存在请新建该文件）：
```Bash
{
  "registry-mirrors": [
    "https://docker.your_domain_name"
  ]
}
```

## 感谢以下项目的付出

- [crproxy](https://github.com/wzshiming/crproxy/tree/master/examples/default)
- [Docker-Proxy](https://github.com/dqzboy/Docker-Proxy)
