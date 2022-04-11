---
layout: post
title: Docker Introduction
tags: Docker
---
## Docker Introduction
### 优势
- 虚拟机：物理机、Hypervisor、多个VM、多个OS、多个APP
- 容器：物理机、操作系统、多个容器、多个APP

1. 每个操作系统都消耗CPU、RAW、存储空间、独立许可证、补丁升级、被攻击的风险
2. 启动时间。容器启动不需要内核，即不需要硬件遍历和初始化等

### Docker镜像

- Docker镜像类似停止的容器（或者类），镜像是构建时结构，容器是运行时结构
- 镜像上启动一个或多个容器，全部停止容器前，镜像无法删除
- 一个镜像可以有多个标签，一个标签只对应一个镜像。新镜像打相同标签，移除旧镜像的标签，旧镜像变成悬虚镜像`<none>:<none>`
```
# 查看本地仓库中镜像
docker image ls
# 拉取至本机
docker image pull <repository>:<tag>
docker image pull <repository>@<sha256>
# 查看本地仓库过滤后的镜像 dangling: true/false; before: 镜像名称或ID之前; since: 镜像名称或ID之后; label: 不显示
docker image ls --filter dangling=true
docker image ls --filter=reference="*:lastest"
# 查看Docker Hub的NAME字段包含nigelpoulton的镜像
docker search nigelpoulton
# 查看本地仓库的镜像散列值
docker image ls --digests <repository>
# 删除本地仓库的镜像，本地拉取的全部镜像ID，-f强制执行
docker image rm <repository>:<tag>
docker image rm $(docker image ls -q) -f
# 查看镜像分层
docker image inspect <repository>:<tag>
# 输出格式
docker image ls --format "{{.Reposity}}: {{.Tag}}: {{.Size}}"
```

### Docker容器

由于容器不运行任何进程则无法存在，当仅有一个进程时，输入exit退出Bash Shell容器会终止。Ctrl-PQ组合则会退出容器但不终止。

```
docker container run <option> <image>:<tag> <app>
# -it参数使容器具备交互性并与终端连接
docker container run -it ubuntu:latest /bin/bash
# 当前系统运行中的容器
docker container ls
# 终端重新连接到运行中的容器，创建一个新的bash进程，此时exit只会退出当前进程，还存在一个进程Docker退出不会终止
docker container exec -it <container-name-or-id> bash
# 停止（支持持久化，docker container ls -a可以查看），启动，暂停，删除容器
docker container stop <container-name-or-id>
docker container start <container-name-or-id>
docker container pause <container-name-or-id>
docker container rm <container-name-or-id>
# Docker daemon重启时，启动容器always与不启动容器unless-stopped，后台、名称、重启策略、端口80为外
docker container run -d --name always -restart always -p 80:8080 <container-name-or-id> sleep 1d
# 查看
docker container inspect <container-name-or-id>
# 删除所有容器
docker container rm $(docker container ls -aq) -f
```

### 打包镜像
```
# 注意最后指定Dockerfile的位置
docker image build -t <tag> <position>
# 查看打镜像过程，SIZE非0则增加层
docker image history <tag>
# 推送至Docker Hub，推送至<Registry>/<Repository>:<Tag>，默认docker.io/<REPO>:<TAG>，但实际上用户并没有docker.io/<REPO>访问权限，推送到<Username>这个二级命名空间下，需要添加标签docker.io/<Username>/<REPO>:<TAG>
# 添加标签
docker image tag <current-tag> <new-tag>
# 推送
docker image push <new-tag>
```
Dockerfile
```
FROM ... AS a
WORKDIR ...
COPY ./pom.xml ./           cp
COPY ./pom.xml ./test.xml   mv
COPY ./conf ./config        mv
RUN ...

FROM ... AS b
WORKDIR ...
COPY --from=a /c/ .
ENTRYPOINT ["java", "-jar", "/d/e.jar"]
CMD ["--spirng.profiles.active=postgres"]
```

1. ENTRYPOINT，一定被执行

- `ENTRYPOINT ["executable", "param1", "param2"]` (exec form, preferred)，`CMD`命令和`docker container run <tag> <param>`可以提供参数
- `ENTRYPOINT command param1 param2` (shell form)，无法其他命令获取参数


2. CMD，优先级低于ENTRYPOINT，参数提供会被`docker container run <tag> <param>`覆盖，CMD多个则**只有最后**一个生效，命令全路径

- `CMD ["executable","param1","param2"]` (exec form, this is the preferred form)
- `CMD ["param1","param2"]` (as default parameters to ENTRYPOINT)
- `CMD command param1 param2` (shell form)

### Docker网络
#### 单机桥接网络

1. `docker network create -d bridge localnet`，Docker引擎和Docker命令
2. docker0网桥，Linux内核和OS工具

- c1容器启动接入localnet网络，`docker container run -d --name c1 --network localnet alpine sleep 1d`
- c2容器启动接入localnet网络，c2容器中ping c1容器成功
- 只能与位于相同网络中的容器通信，**端口映射**解决，`docker container run -d --name web --network localnet --publish 5000:80 nginx`，缺点5000**端口被占用**

#### 多机覆盖网络
1. 构建Swarm
```
docker swarm init --advertise-addr=<ip> --listen-addr=<ip>:<port>
docker swarm join --token xx <listen-addr>
```
2. 创建覆盖网络，管理节点上
```
docker network create -d overlay uber-net
```

3. Docker服务（两个容器）连接覆盖网络，node2上可以看到uber-net网络
```
docker service create --name test --network uber-net --replicas 2 ubuntu sleep infinity
```

4. 查看容器ip
```
docker network inspect uber-net
docker container ls
docker container inspect --format='{{range .NetworkSettings.Networks}}{}{{.IPAddress}}{{end}}' <container-id>
```

#### 接入现有网络
Macvlan无需端口映射或额外桥接，主机接口直接访问容器接口；但是主机网卡需要为**混杂模式**，公有云平台不允许。
```
# 在子接口eth0.100上面创建macvlan100
docker network create -d macvlan --subnet=10.0.0.0/24 --ip-range=10.0.0.0/25 --gateway=10.0.0.1 -o parent=eth0.100 macvlan100
# 启动
docker container run -d --name c1 --network macvlan100 alpine sleep 1d
```
#### Swarm发布模式
1. Ingress（默认），任意节点，存在副本则任意一个，流量均分
2. Host，`--replicas 2 --publish published=5000,target=80,mode=host`，服务副本所在节点可以访问

### Docker持久化
```
docker volume create myvol
docker volume ls
docker volume inspect myvol
# 删除未装入容器的所有卷
docker volume prune
docker volume rm myvol
# bizvol卷不存在会被创建
docker container run -dit --name vol --mount source=bizvol,target=/vol alpine
```