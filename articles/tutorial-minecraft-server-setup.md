---
title: 用Docker部署Paper Minecraft服务器
date: 2026-10-18
summary: 本教程详细阐述如何在服务器上，在Ubuntu上通过Docker部署并管理基于Paper的原版Minecraft服务器。
tags: [ 教程, Linux, Docker ]
cover: /images/articles/minecraft-screenshot-1.png
---


> 本教程假设您有基本网络知识和Linux基础，并且对Docker和Minecraft有基本了解。
>
>本教程使用纯命令行操作，不使用开服面板等辅助工具，目标是部署一台基于Paper核心、开启正版验证的Minecraft服务端。
>
>在完成之后，您应该可以在Docker容器中运行一个Minecraft服务器并成功访问它。
>
>本文使用人脑生成。

## 准备工作

首先，您需要一台具有公网IP的主机，并安装Ubuntu
Linux。云服务器一般都会分配公网IP地址并预装Linux操作系统。如果您没有云服务器，也可以使用内网穿透或组建虚拟局域网。本教程默认您拥有一台云服务器。

### 登录远程主机

您可以使用Windows自带终端或Xshell等工具，通过SSH远程访问服务器。请确保服务器已安装SSH，并开放对应端口（默认22）。

使用以下命令登录远程服务器的root账户

```bash
ssh root@你的服务器地址
```

输入密码后，您应该可以看到终端中的提示符。

```bash
root@yourhostname:~#
```

### 安装Docker

在开始教程之前，请在服务器中安装Docker与Docker
Compose。本教程不赘述此过程。值得提醒的是，常见的云服务器厂商（如阿里云、腾讯云等）通常会提供镜像加速服务以加速常见镜像的拉取。但有些镜像不会被加速，对于这种情况，您可以使用公共镜像源。我一般使用：

```bash
https://docker.m.daocloud.io
```

### 下载Paper服务器核心

接着，请回到您的日常工作电脑，访问[Paper | PaperMC](https://papermc.io/downloads/paper)下载您所需版本的Paper服务器核心。点击下载后，您会得到一个
`.jar`结尾的文件，它就是我们需要的服务器核心。

> Minecraft服务器核心，也叫服务端，简单说就是**运行在服务器上、负责处理游戏逻辑的程序**
> 。没有它，服务器就无法启动。不同核心决定了服务器支持什么功能（插件、Mod等）以及性能如何。
>
>您也可以从[Minecraft官网](minecraft.net)
> 下载原版（Vanilla）服务器核心部署。不过Paper服务器核心相比原版，包含了大量性能优化，稳定性和性能更好，且支持插件，Paper服务器几乎已经成为了原版插件服的事实标准。

### 初始化服务器核心

请创建一个文件夹，命名为您喜欢的名字（我命名为`paper-server`），再在其中创建一个名为`server`的文件夹，并将您的服务端核心文件放进去，重命名为
`server.jar`。这一步是为了确保命名规范性，避免后续可能的错误。

然后，打开您电脑上的命令提示符，并确保您安装了您游戏所需要的Java版本。我以Windows操作系统为例，但Linux上的流程都差不多。请输入：

```bash
cd 您server.jar所在的文件路径
```

这个命令将终端的工作目录跳转到服务器核心所在的地方。您也可以在文件资源管理器的路径地址直接输入`cmd`，这会打开命令提示符并自动跳转。接着，输入：

```bash
java -jar server.jar
```

如果您安装了正确版本的Java，服务端将开始运行并跳出一大堆东西。接着，服务器会因为没有同意EULA协议而停止运行。按Ctrl+C停止终端，你可以注意到服务器核心所在的文件夹内多出了一大堆东西。我们只需要关注两个文件：
`eula.txt`和`server.properties`。

> `eula.txt`表示您是否同意Mojang的EULA协议；`server.properties`是服务器的主要配置文件。

首先，将`eula.txt`的内容编辑为`eula=true`。

接着，打开`server.properties`，您可以看到一大串配置项。目前，您只需要按需求调整这几个配置项：

```properties
# RCON的密码以及暴露的端口
rcon.password= 
rcon.port=25575

# 服务器暴露的端口
server-port=25565

# 最大的玩家数，请按照您的服务器性能以及需求调整
max-players=8

# 您服务器的名称
motd=

# 是否开启正版验证
white-list=true
```

> RCON（远程控制台）是基于TCP/IP的远程控制协议，主要用于游戏服务器管理。后续，您可以通过RCON工具对服务器进行管理。

在本教程中，我们保持各项端口不变，并填写`rcon.password`。`motd`是选填项，您可以填一个您喜欢的名称。

到此，您已经完成了服务器核心的初始化。

### 编写部署脚本

我们需要三个文件来完成自动化部署：

1. 启动脚本`start.sh`
2. 镜像构建文件`Dockerfile`
3. Docker Compose文件`docker-compose.yml`

#### 启动脚本

其实就是一段bash命令，让Java把服务端核心跑起来。**请将这个文件放在服务器核心所处的同级目录内**。

基础命令是这样的：

```bash
# 让Java可执行文件以命令行形式启动服务器核心
java -jar server.jar --nogui
```

不过，我们一般会使用被称 为`Aikar's Flags`的Java虚拟机参数来跑服务端，它包含了大量调优。

```bash
java -Xms4G -Xmx4G -XX:+UseG1GC -XX:+ParallelRefProcEnabled -XX:MaxGCPauseMillis=200 -XX:+UnlockExperimentalVMOptions -XX:+DisableExplicitGC -XX:+AlwaysPreTouch -XX:G1HeapWastePercent=5 -XX:G1MixedGCCountTarget=4 -XX:InitiatingHeapOccupancyPercent=15 -XX:G1MixedGCLiveThresholdPercent=90 -XX:G1RSetUpdatingPauseTimePercent=5 -XX:SurvivorRatio=32 -XX:+PerfDisableSharedMem -XX:MaxTenuringThreshold=1 -Dusing.aikars.flags=https://mcflags.emc.gs -Daikars.new.flags=true -XX:G1NewSizePercent=40 -XX:G1MaxNewSizePercent=50 -XX:G1HeapRegionSize=16M -XX:G1ReservePercent=15 -jar server.jar --nogui
```

那堆参数是干嘛的不用管。您只需要根据您的服务器配置调整`-Xms -Xmx`
这两项就行了。这是虚拟机的内存设置，它需要一个整数值和单位。将两个参数设置为相同值可避免JVM频繁申请额外内存导致的卡顿。对于4人左右的原版Minecraft服务器，2-3G的内存就已足够。

> 请注意，这个内存大小是专门分配给JVM的，操作系统、Docker等其它服务本身还会占用一部分内存，请务必留出余量。

#### Dockerfile

回到上级目录，也就是`server`这个文件夹的同级目录，`paper-server`文件夹下，请创建一个文本文件，并命名为`Dockerfile`（*
*不带`.txt`扩展名**）。在其中编辑这样的内容：

```dockerfile
FROM eclipse-temurin:25-jdk-alpine

# 设置工作目录
WORKDIR /server

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && \
        apk add --no-cache tini curl bash

COPY /server /server
RUN find /server -name "*.sh" -exec chmod +x {} \;

EXPOSE 25565

ENTRYPOINT ["/sbin/tini", "--"]

CMD ["./start.sh"]
```

在这个Dockerfile里，我们拉取轻量版Java 25的Docker镜像，并设置容器内部的工作目录为`/server`，通过阿里云的镜像拉取`tini`、
`curl`、`bash`等工具。

随后，我们将Dockerfile所在目录中的`/server`文件夹整体复制到工作目录中，并给其中的`start.sh`赋予可执行权限。

接着，我们暴露`25565`端口，也就是您在`server.properties`中设置的端口，使用`tini`作为初始化进程，并运行`start.sh`
，把Minecraft服务器跑起来。

#### docker-compose.yml

编辑完`Dockerfile`后，请在`Dockerfile`的同级目录创建一个`docker-compose.yml`的文本文件，编辑其中的内容：

```yaml
services:
  minecraft:
    build: .
    container_name: minecraft-server
    ports:
      - "25565:25565"
      - "25575:25575"
    volumes:
      # 挂载服务器文件夹
      - /workspace/server:/server
    restart: unless-stopped
```

这个文件让Docker Compose执行同级的Dockerfile以构建镜像，并将容器名称命名为`minecraft-server`，暴露`25565 25575`
两个端口，也就是服务器端口和RCON端口。

Docker Compose会将您服务器内的`/workspace/server`挂载到容器内的工作目录内。

到此，准备工作就结束了。此时，您的文件结构应该是这样的：

```text
paper-server/
├── docker-compose.yml
├── Dockerfile
└── server/
    ├── .../
    ├── eula.txt
    ├── server.jar
    ├── server.properties
    └── start.sh
```

## 开始部署

### 上传文件

首先，请将您的`paper-server`文件夹整体打成一个`.zip`压缩包。

接着，使用SSH，以root用户登录远程主机，在根目录创建一个`/workspace/server`文件夹。

```bash
mkdir -p /workspace/server
```

最好也在您账户对应的工作目录内也创建一个文件夹，比如`uploads/`。我们使用`scp`命令将`paper-server.zip`从您的工作电脑上传到服务器中。

```bash
scp paper-server.zip root@yourhostname:/home/username/uploads/
```

等待上传完毕，回到服务器终端中，跳转到上传目录，看看上传成功了没：

```bash
cd /home/username/uploads/
ll
```

您应该能看到显示了`paper-server.zip`，然后使用`unzip`工具解压这个压缩包(请确保安装了`unzip`)。

```bash
unzip paper-server.zip
```

它会将压缩包解压为同名的目录，并放在相同的目录下。

### 运行服务器

我们进入这个目录。

```bash
cd paper-server/
```

然后，用Docker把容器跑起来。

```bash
docker compose up -d
```

Docker会自动拉取需要的镜像并构建您的Minecraft服务器镜像，然后运行它的容器。这需要一点时间，您可以去倒杯咖啡。

> 如果您发现错误并修改了`paper-server/`中的代码，您可以使用`docker compose up -d --build`重新构建镜像。

使用`docker ps -a`查看所有构建完毕的容器：

```bash
docker ps -a

# 示例输出
CONTAINER ID   IMAGE                    COMMAND                  CREATED       STATUS        PORTS                                                                                              NAMES
a94abfb6e647   paper-server-minecraft   "/sbin/tini -- ./sta…"   6 weeks ago   Up 16 hours   0.0.0.0:25565->25565/tcp, [::]:25565->25565/tcp, 0.0.0.0:25575->25575/tcp, [::]:25575->25575/tcp   minecraft-server
```

接下来，使用以下命令查看Minecraft服务器输出的日志：

```bash
docker logs -f minecraft-server
```

终端会跳出来一大堆东西，不用管，注意看有没有这一行：

```
Done (XX.XXXs)! For help, type "help"
```

出现了这一行，服务器就已经成功运行了，为了让玩家可以连接到服务器，您还需要配置防火墙以暴露服务器端口。Ubuntu使用ufw作为防火墙。

```bash
sudo ufw allow 25565/tcp # 暴露TCP协议的25565端口
```

接着，您就可以打开Minecraft客户端，在多人游戏中添加服务器，并输入您服务器的公网IP地址和端口号：

```
IP地址:端口号
```

不出问题，您就可以成功进入服务器了！

---

## 后续

### 使用RCON管理服务器

默认情况下，登录的任何玩家都是没有OP权限的。您需要安装RCON客户端以远程执行命令。具体过程本教程并不阐述。一般来说，将RCON安装在服务器内就已足够。

### 备份世界

养成定时备份世界存档的习惯有益无害。本教程的存档文件在`/workspace/server/world`这个文件夹下。您也可以写一个脚本来定时备份。

### 安装插件

Paper服务端支持大量插件。若您不使用插件管理器，则需要在安装插件时停止服务器，或安装完毕后重启服务器。

```bash
# 停止服务器
docker stop minecraft-server
# 重启服务器
docker restart minecraft-server
```

将插件的`.jar`文件放在`/workspace/server/plugin`下即可。

---

非常感谢您看到这里！
