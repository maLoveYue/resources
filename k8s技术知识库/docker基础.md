# 1. docker安装

```shell
Uninstall old versions
#sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
Set up the repository
#sudo yum install -y yum-utils
#sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
 
Install Docker Engine
#sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
Start Docker.
#sudo systemctl start docker
```

如果配置docker加速器？？

# 2.docker基础命令

## 2.1 docker version

查看docker版本,分为client 端和server端

```shell
[root@k8s-master01 docker]# docker version
Client: Docker Engine - Community
 Version:           24.0.7
 API version:       1.40 (downgraded from 1.43)
 Go version:        go1.20.10
 Git commit:        afdd53b
 Built:             Thu Oct 26 09:11:35 2023
 OS/Arch:           linux/amd64
 Context:           default

Server: Docker Engine - Community
 Engine:
  Version:          19.03.15
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.13.15
  Git commit:       99e3ed8919
  Built:            Sat Jan 30 03:16:33 2021
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.24
  GitCommit:        61f9fd88f79f081d64d6fa3bb1a0dc71ec870523
 runc:
  Version:          1.1.9
  GitCommit:        v1.1.9-0-gccaecfc
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683

```

* OCI: Open Contanier Initiative的简称，由Linux基金会主导开发OCI规范和标准，目的是围绕容器格式和Runtime(运行时)制定的一个工业化标准。
* Containerd: Docker为了兼容OCI标准，将容器Runtime及其管理功能从Docker守护进程中剥离出来，用于不启动Docker也能直接通过Containerd来管理。
* RunC: Docker按照OCF(Open Container FOrmat)开放容器格式标准制定的一个轻量级工具，可以使用RunC不通过Docker引擎即可实现容器的启动、停止和资源隔离等功能。

## 2.2 docker info

```shell

[root@k8s-master01 docker]# docker info
Client: Docker Engine - Community
 Version:    24.0.7
 Context:    default
 Debug Mode: false
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.11.2
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v2.21.0 
    Path:     /usr/libexec/docker/cli-plugins/docker-compose

Server:
 Containers: 13 #机器上运行的容器数量
  Running: 6 #运行的容器据量
  Paused: 0 #暂停的容器据量
  Stopped: 7 #停止的容器据量
 Images: 6 #镜像数据量
 Server Version: 19.03.15
 Storage Driver: overlay2 #存储驱动，一般为overlay2 性能好速度快
  Backing Filesystem: xfs #服务器文件系统
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file #日志本地存储
 Cgroup Driver: systemd
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 61f9fd88f79f081d64d6fa3bb1a0dc71ec870523
 runc version: v1.1.9-0-gccaecfc
 init version: fec3683
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 4.19.12-1.el7.elrepo.x86_64
 Operating System: CentOS Linux 7 (Core)
 OSType: linux
 Architecture: x86_64
 CPUs: 2
 Total Memory: 3.829GiB
 Name: k8s-master01
 ID: 6L7V:TEWA:3PV7:5KDR:ZGU5:UPOY:R2D5:E4CJ:UN47:F4FV:AROP:ICRZ
 Docker Root Dir: /var/lib/docker #根目录，建议该目录独立一块ssd盘
 Debug Mode: false
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false #生产环境建议设置true，热更新，重启docker后不会重启所有的容器

```

## 2.3 搜索镜像

```shell
[root@harbor ~]# docker search nginx
NAME                               DESCRIPTION                                     STARS     OFFICIAL   AUTOMATED
nginx                              Official build of Nginx.                        19268     [OK]
unit                               Official build of NGINX Unit: Universal Web …   17        [OK]

```

## 2.4 拉取/下载镜像

```shell
#docker pull nginx:版本号 不指定默认latest
[root@harbor ~]# docker pull nginx
Using default tag: latest
latest: Pulling from library/nginx
1f7ce2fa46ab: Pull complete
9b16c94bb686: Pull complete
9a59d19f9c5b: Pull complete
9ea27b074f71: Pull complete
c6edf33e2524: Pull complete
84b1ff10387b: Pull complete
517357831967: Pull complete
Digest: sha256:10d1f5b58f74683ad34eb29287e07dab1e90f10af243f151bb50aa5dbb4d62ee
Status: Downloaded newer image for nginx:latest
docker.io/library/nginx:latest

```

## 2.5 更改版本号

```shell
#docker tag nginx 192.168.0.203:8089/mark/nginx
更改镜像tag不会占用空间，镜像层都是一样的
```



## 2.6 镜像拉取和上传

```
 上传到镜像仓库
#docker push 192.168.0.203:8089/mark/nginx

拉取镜像
#docker pull 192.168.0.203:8089/mark/nginx

```

## 2.7 镜像仓库登陆

```SHELL
[root@harbor harbor]#  docker login http://192.168.0.203:8089
Authenticating with existing credentials...
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded

```

## 2.8 启动容器

### 2.8.1 前台启动

```shell
# docker run -it --rm 192.168.0.203:8089/mark/nginx
 --rm，前台退出后，删除容器,适合调试
```



### 2.8.2  后台启动

```shell
# docker run -itd 192.168.0.203:8089/mark/nginx 
1a7684d14f8a9919631e3eb92943d3e04a40eca1fe83804d0bf24f9686434bac
```



### 2.8.3 暴露端口

```shell
暴露容器端口
# docker run -itd   -p 111:80 192.168.0.203:8089/mark/nginx
本机的111端口映射到容器的80

[root@harbor harbor]# curl 192.168.0.203:111
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
```

## 2.9 查看、停止、删除容器

```shell
删除
#docker rm 容器id -f是强制删除
停止
#docker stop 容器id
查看运行的容器
#docker ps 
查看所有容器
#docker ps -a
查看运行容器的id
#docker ps -q
查看容器详细信息
docker inspect 容器id
```

## 2.10 本地持久化

```shell
本地化的优势数据不会随着容器的变化而变化
#docker run -itd -p 222:80 -v /mnt/nginxhtml:/usr/share/nginx/html 192.168.0.203:8089/mark/nginx
20189082e9814b62c52b6714053fd66505f601dc2c8ec24a64506219b301f871
[root@harbor nginxhtml]# echo "test" index.html
test index.html
[root@harbor nginxhtml]# curl  http://192.168.0.203:222
test
```

## 2.11 查看镜像层构建信息

```shell
[root@harbor data]# docker history a6bd71f48f68
IMAGE          CREATED      CREATED BY                                      SIZE      COMMENT
a6bd71f48f68   7 days ago   /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon…   0B
<missing>      7 days ago   /bin/sh -c #(nop)  STOPSIGNAL SIGQUIT           0B
<missing>      7 days ago   /bin/sh -c #(nop)  EXPOSE 80                    0B
<missing>      7 days ago   /bin/sh -c #(nop)  ENTRYPOINT ["/docker-entr…   0B
<missing>      7 days ago   /bin/sh -c #(nop) COPY file:9e3b2b63db9f8fc7…   4.62kB
<missing>      7 days ago   /bin/sh -c #(nop) COPY file:57846632accc8975…   3.02kB
<missing>      7 days ago   /bin/sh -c #(nop) COPY file:3b1b9915b7dd898a…   298B
<missing>      7 days ago   /bin/sh -c #(nop) COPY file:caec368f5a54f70a…   2.12kB
<missing>      7 days ago   /bin/sh -c #(nop) COPY file:01e75c6dd0ce317d…   1.62kB
<missing>      7 days ago   /bin/sh -c set -x     && groupadd --system -…   112MB
<missing>      7 days ago   /bin/sh -c #(nop)  ENV PKG_RELEASE=1~bookworm   0B
<missing>      7 days ago   /bin/sh -c #(nop)  ENV NJS_VERSION=0.8.2        0B
<missing>      7 days ago   /bin/sh -c #(nop)  ENV NGINX_VERSION=1.25.3     0B
<missing>      7 days ago   /bin/sh -c #(nop)  LABEL maintainer=NGINX Do…   0B
<missing>      7 days ago   /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>      7 days ago   /bin/sh -c #(nop) ADD file:d261a6f6921593f1e…   74.8MB

```



# 3. Dockefile

## 3.1  Dockfile的常用指令

| 指令名称   | 描述                                                         | 备注                                                         |
| :--------- | :----------------------------------------------------------- | ------------------------------------------------------------ |
| FROM       | FROM <IMAGE> 或FROM <IMAGE>:<TAG> 指定基础镜像               | 基础镜像选择alpine                                           |
| LABEL      | LABEL key=value key=value version="1.0" maintainer="mark"    |                                                              |
| RUN        | RUN <command>   RUN [“/bin/bash”, “-c”, “echo hello”]        | 多条命令写在一起                                             |
| EXPOSE     | EXPOSE <PORT>                                                | 使用这个指令的目的是告诉应用程序容器内应用程序会使用的端口，在运行时还需要使用-p参数指定映射端口。 |
| CMD        | CMD <shell 命令> CMD [ "executable", "param1", "param2" ] (exec模式) | 会被启动命令覆盖["echo", "hello"],如果有多个CMD只有最后一个会被执行 |
| ENTRYPOINT | ENTRYPOINT <command> (shell模式) ENTRYPOINT [ "executable", "param1", "param2" ] (exec模式) | 不会被覆盖，如果想覆盖ENTRYPOINT，则需要在`docker run`中指定`--entrypoint`选项 |
| ADD        | ADD <src> <dest> ADD ["<src>" "<dest>"]                      | 会自动解压只针对tar.gz压缩包，如果复制目录，目标端要加上目录名；ADD tmpe/ /root/tmpe |
| COPY       | COPY <src> <dest>                                            | 不会自动解压                                                 |
| WORKDIR    | WORKDIR <dirname> 工作目录                                   | 容器运行时会自动切到该目录                                   |
| ENV        | ENV <KEY> <VALUE> ENV <KEY>=<VALUE>                          | 环境变量                                                     |
| USER       | USER nginx                                                   | 运行时的用户，需要提前创建                                   |
| ARG        | 动态构建传参 docker build --build-arg arg=xxx .              |                                                              |

## 3.2 Dockfile编写

```YAML
FROM centos:7
LABEL maintainer="mark" version="1.0"
ARG NAME
CMD ["echo", "hello" ,"123"]
ENTRYPOINT ["echo", "LLL"]
ADD txt.tar.gz /tmp
COPY tmep /tmp/tmep
WORKDIR /home/mk
ENV A=1 NAME="MARK"
RUN mkdir $NAME

```

## 3.3 Dockerfile优化 （重点）

* 多阶段构建

```
#先完成构建然后对镜像命名
FROM golang:1.14.4-alpine AS builder
LABEL maintainer="mark" version="1.0"
WORKDIR /opt
COPY hw.go /opt
RUN go build /opt/hw.go

#生成镜像
FROM alpine
#如果不命名builder 第一FROM为0，以此类推 COPY --from=0 /opt/hw .
COPY --from=builder /opt/hw .
CMD ["./hw"]


[root@harbor dockfiles]# docker images
REPOSITORY                                                        TAG          IMAGE ID       CREATED              SIZE
hw                                                                two          17227cd52043   About a minute ago   9.4MB
hw                                                                one          e67643f7f2d1   4 minutes ago        372MB

```



Dockerfile问题排查





# 4. harbor安装和配置

## 4.1 harbor介绍

Harbor 是由 VMware 公司中国团队为企业用户设计的 Registry server 开源项目，包括了权限管理(RBAC)、LDAP、审计、管理界面、自我注册、HA 等企业必需的功能，同时针对中国用户的特点，设计镜像复制和中文支持等功能。

* 基于角色的访问控制 - 用户与 Docker 镜像仓库通过“项目”进行组织管理，一个用户可以对多个镜像仓库在同一命名空间（project）里有不同的权限。

* 镜像复制 - 镜像可以在多个 Registry 实例中复制（同步）。尤其适合于负载均衡，高可用，混合云和多云的场景。

* 图形化用户界面 - 用户可以通过浏览器来浏览，检索当前 Docker 镜像仓库，管理项目和命名空间。

* AD/LDAP 支持 - Harbor 可以集成企业内部已有的 AD/LDAP，用于鉴权认证管理。

* 审计管理 - 所有针对镜像仓库的操作都可以被记录追溯，用于审计管理。

* 国际化 - 已拥有英文、中文、德文、日文和俄文的本地化版本。更多的语言将会添加进来。

* RESTful API - RESTful API 提供给管理员对于 Harbor 更多的操控, 使得与其它管理软件集成变得更容易。

* 部署简单 - 提供在线和离线两种安装工具， 也可以安装到 vSphere 平台(OVA 方式)虚拟设备。
  

## 4.2 harbor安装部署

**基础准备**：harbor部署依赖docker和docker-compose

### 4.2.1 docker安装

参考步骤1

### 4.2.2 docker-compose

* 下载docker-compose安装包

  ```shell
  #curl -L "https://github.com/docker/compose/releases/download/2.23.1/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose    
  
  #chmod +x /usr/local/bin/docker-compose
  
  #docker-compose version   # 查看版本信息
  ```

  

### 4.2.3  harbor

#### 1.下载harbor离线包 

https://github.com/goharbor/harbor/releases/download/v2.9.1/harbor-offline-installer-v2.9.1.tgz

#### 2.解压安装包

```
tar -zxf harbor-offline-installer-v2.9.1.tgz -C /usr/local/
```

#### 3.生成https证书

```shell
生成key
# openssl genrsa -out ca.key 4096

生成证书，www.harbor.com为harbor的域名，也可以为证书
#openssl req -x509 -new -nodes -sha512 -days 3650  -subj "/CN=www.harbor.com"  -key ca.key  -out ca.crt
 创建私钥
 #openssl genrsa -out server.key 4096
 
生成证书签名请求
 # openssl req  -new -sha512  -subj "/CN=www.harbor.com"  -key server.key  -out server.csr
 # ll
total 20
-rw-r--r-- 1 root root 1801 Nov 24 19:43 ca.crt
-rw-r--r-- 1 root root 3243 Nov 24 19:42 ca.key
-rw-r--r-- 1 root root 1590 Nov 24 19:48 server.csr
-rw-r--r-- 1 root root 3243 Nov 24 19:47 server.key

生成harbor仓库主机的证书
#cat > v3.ext <<EOF
 authorityKeyIdentifier=keyid,issuer
 basicConstraints=CA:FALSE
 keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
 extendedKeyUsage = serverAuth 
 subjectAltName = @alt_names
 [alt_names]
 DNS.1=www.harbor.com
EOF

生成harbor仓库主机的证书
# openssl x509 -req -sha512 -days 3650 -extfile v3.ext -CA ca.crt -CAkey ca.key -CAcreateserial -in server.csr -out server.crt

 #导入配置
 #root@eb7023:/root/harbor>./prepare 
 prepare base dir is set to /root/harbor
 Clearing the configuration file: /config/log/logrotate.conf
 Clearing the configuration file: /config/log/rsyslog_docker.conf
 Clean up the input dir
 ##停止当前运行的harbor
 #root@eb7023:/root/harbor>docker-compose down -v
 /usr/lib/python2.7/site-packages/paramiko/transport.py:33: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in a future release.
   from cryptography.hazmat.backends import default_backend
 Stopping harbor-jobservice ... done
 Removing network harbor_harbor
 ##后台运行的harbor
 #root@eb7023:/root/harbor>docker-compose up -d
 /usr/lib/python2.7/site-packages/paramiko/transport.py:33: CryptographyDeprecationWarning: Python 2 is no longer supported by the Python core team. Support for it is now deprecated in cryptography, and will be removed in a future release.
 Creating harbor-jobservice ... done
 Creating nginx             ... done
 
 将server证书cp到docker所在的机器固定目录中
# root@eb7023:/root/harbor>cd /etc/docker/certs.d/
#mkdir -p /etc/docker/certs.d/harbor23.com      
# root@eb7023:/etc/docker/certs.d>cd /data/certs/
# root@eb7023:/data/certs>cp server.crt  /etc/docker/certs.d/harbor23.com/server.crt

登陆测试
[root@harbor harbor.com]# docker login www.harbor.com
Username: admin
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store
Login Succeeded
```

#### 4.修改配置文件

```shell
#cd /usr/local/harbor/
#cp harbor.yml.tmp harbor.yml
#vim harbor.yml

hostname: 192.168.10.107                 # 修改为本机IP

#https:                                 # 因为自己搭建未使用 https 所以注释
#  # https port for harbor, default is 443
#  port: 443
#  # The path of cert and key files for nginx
#  certificate: /your/certificate/path
#  private_key: /your/private/key/path
harbor_admin_password: Harbor12345      # admin 密码
```



#### 5.执行安装脚本

```shell
#./install.sh
......
Building with native build. Learn about native build in Compose here: https://docs.docker.com/go/compose-native-build/
Creating network "harbor_harbor" with the default driver
Creating harbor-log ... done
Creating registryctl   ... done
Creating harbor-db     ... done
Creating redis         ... done
Creating registry      ... done
Creating harbor-portal ... done
Creating harbor-core   ... done
Creating harbor-jobservice ... done
Creating nginx             ... done
✔ ----Harbor has been installed and started successfully.----
```

## 4.3 harbor服务开机自启

* 创建harbor.service

  ```shell
  root@harbor system]# vim /lib/systemd/system/harbor.service
  [Unit]
  Description=Harbor
  After=docker.service systemd-networkd.service systemd-resolved.service
  Requires=docker.service
  Documentation=http://github.com/vmware/harbor
  
  [Service]
  Type=simple
  Restart=on-failure
  RestartSec=5
  ExecStart=/usr/local/bin/docker-compose -f  /opt/harbor/docker-compose.yml up
  ExecStop=/usr/local/bin/docker-compose -f /opt/harbor/docker-compose.yml down
  
  [Install]
  WantedBy=multi-user.target
  ```

  

  * 设置开机自启

  ```shell
  # systemctl enable harbor
  Created symlink from /etc/systemd/system/multi-user.target.wants/harbor.service to /usr/lib/systemd/system/harbor.service.
  # systemctl start  harbor
  ```

  

  

