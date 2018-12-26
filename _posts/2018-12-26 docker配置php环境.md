# docker配置php环境

标签（空格分隔）： php docker lnmp

---

### 一、docker基本概念

#### 1. 基本介绍
    Docker 是基于 Linux 内核的 cgroup，namespace，以及 AUFS 类的 Union FS 等技术，对进程进行封装隔离，属于操作系统层面的虚拟化技术。
    
    传统虚拟机技术是虚拟出一套硬件后，在其上运行一个完整操作系统，在该系统上再运行所需应用进程；而容器内的应用进程直接运行于宿主的内核，容器内没有自己的内核，而且也没有进行硬件虚拟。因此容器要比传统虚拟机更为轻便。

#### 2. docker容器
    镜像（Image）和容器（Container）的关系，就像是面向对象程序设计中的 类 和 实例 一样，镜像是静态的定义，容器是镜像运行时的实体。容器可以被创建、启动、停止、删除、暂停等。
    
    容器的实质是进程，但与直接在宿主执行的进程不同，容器进程运行于属于自己的独立的 命名空间。因此容器可以拥有自己的 root 文件系统、自己的网络配置、自己的进程空间，甚至自己的用户 ID 空间。容器内的进程是运行在一个隔离的环境里，使用起来，就好像是在一个独立于宿主的系统下操作一样。这种特性使得容器封装的应用比直接在宿主运行更加安全。也因为这种隔离的特性，很多人初学 Docker 时常常会混淆容器和虚拟机。
    
    前面讲过镜像使用的是分层存储，容器也是如此。每一个容器运行时，是以镜像为基础层，在其上创建一个当前容器的存储层，我们可以称这个为容器运行时读写而准备的存储层为容器存储层。
    
    容器存储层的生存周期和容器一样，容器消亡时，容器存储层也随之消亡。因此，任何保存于容器存储层的信息都会随容器删除而丢失。
    
    按照 Docker 最佳实践的要求，容器不应该向其存储层内写入任何数据，容器存储层要保持无状态化。所有的文件写入操作，都应该使用 数据卷（Volume）、或者绑定宿主目录，在这些位置的读写会跳过容器存储层，直接对宿主（或网络存储）发生读写，其性能和稳定性更高。
    
    数据卷的生存周期独立于容器，容器消亡，数据卷不会消亡。因此，使用数据卷后，容器删除或者重新运行之后，数据却不会丢失。

#### 3. docker镜像
    我们都知道，操作系统分为内核和用户空间。对于 Linux 而言，内核启动后，会挂载 root 文件系统为其提供用户空间支持。而 Docker 镜像（Image），就相当于是一个 root 文件系统。比如官方镜像 ubuntu:16.04 就包含了完整的一套 Ubuntu 16.04 最小系统的 root 文件系统。

    Docker 镜像是一个特殊的文件系统，除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。

#### 4. 分层存储
    因为镜像包含操作系统完整的 root 文件系统，其体积往往是庞大的，因此在 Docker 设计时，就充分利用 Union FS 的技术，将其设计为分层存储的架构。所以严格来说，镜像并非是像一个 ISO 那样的打包文件，镜像只是一个虚拟的概念，其实际体现并非由一个文件组成，而是由一组文件系统组成，或者说，由多层文件系统联合组成。
    
    镜像构建时，会一层层构建，前一层是后一层的基础。每一层构建完就不会再发生改变，后一层上的任何改变只发生在自己这一层。比如，删除前一层文件的操作，实际不是真的删除前一层的文件，而是仅在当前层标记为该文件已删除。在最终容器运行的时候，虽然不会看到这个文件，但是实际上该文件会一直跟随镜像。因此，在构建镜像的时候，需要额外小心，每一层尽量只包含该层需要添加的东西，任何额外的东西应该在该层构建结束前清理掉。
    
    分层存储的特征还使得镜像的复用、定制变的更为容易。甚至可以用之前构建好的镜像作为基础层，然后进一步添加新的层，以定制自己所需的内容，构建新的镜像。

#### 5. docker 仓库
    镜像构建完成后，可以很容易的在当前宿主机上运行，但是，如果需要在其它服务器上使用这个镜像，我们就需要一个集中的存储、分发镜像的服务，Docker Registry 就是这样的服务。
    
    一个 Docker Registry 中可以包含多个仓库（Repository）；每个仓库可以包含多个标签（Tag）；每个标签对应一个镜像。
    
    通常，一个仓库会包含同一个软件不同版本的镜像，而标签就常用于对应该软件的各个版本。我们可以通过 <仓库名>:<标签> 的格式来指定具体是这个软件哪个版本的镜像。如果不给出标签，将以 latest 作为默认标签。
    
    以 Ubuntu 镜像 为例，ubuntu 是仓库的名字，其内包含有不同的版本标签，如，14.04, 16.04。我们可以通过 ubuntu:14.04，或者 ubuntu:16.04 来具体指定所需哪个版本的镜像。如果忽略了标签，比如 ubuntu，那将视为 ubuntu:latest。
    
    仓库名经常以 两段式路径 形式出现，比如 jwilder/nginx-proxy，前者往往意味着 Docker Registry 多用户环境下的用户名，后者则往往是对应的软件名。但这并非绝对，取决于所使用的具体 Docker Registry 的软件或服务。

### 二、实践

#### 1. 安装
具体安装细节根据自身系统版本去参考官网。
这里为了方便采用脚本自动安装（ubuntu）:
```bash
$ curl -fsSL get.docker.com -o get-docker.sh
$ sudo sh get-docker.sh --mirror Aliyun
```
执行这个命令后，脚本就会自动的将一切准备工作做好，并且把 Docker CE 的 Edge 版本安装在系统中。

启动 Docker CE
```bash
$ sudo systemctl enable docker
$ sudo systemctl start docker
```
Ubuntu 14.04 请使用以下命令启动：
```bash
$ sudo service docker start
```
测试 Docker 是否安装正确
```bash
$ docker run hello-world

Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
ca4f61b1923c: Pull complete
Digest: sha256:be0cd392e45be79ffeffa6b05338b98ebb16c87b255f48e297ec7f98e123905c
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://cloud.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/
```
若能正常输出以上信息，则说明安装成功。

镜像加速
对于使用 systemd (Ubuntu 16.04+、Debian 8+、CentOS 7) 的系统，请在 /etc/docker/daemon.json 中写入如下内容（如果文件不存在请新建该文件）
```js
{
  "registry-mirrors": [
    "https://registry.docker-cn.com"
  ]
}
```
之后重新启动服务。
```bash
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

<br>
#### 2. 获取镜像 操作容器
[Docker Hub](https://hub.docker.com/explore/) 上有大量的高质量的镜像可以用，下面就从获取这些镜像。

从 Docker 镜像仓库获取镜像的命令是 docker pull。其命令格式为：
    
    docker pull [选项] [Docker Registry 地址[:端口号]/]仓库名[:标签]

具体的选项可以通过 docker pull --help 命令看到，这里我们说一下镜像名称的格式。

+ Docker 镜像仓库地址：地址的格式一般是 <域名/IP>[:端口号]。默认地址是 Docker Hub。
+ 仓库名：如之前所说，这里的仓库名是两段式名称，即 <用户名>/<软件名>。对于 Docker Hub，如果不给出用户名，则默认为 library，也就是官方镜像。

比如：

    $ docker pull ubuntu:16.04
    16.04: Pulling from library/ubuntu
    bf5d46315322: Pull complete
    9f13e0ac480c: Pull complete
    e8988b5b3097: Pull complete
    40af181810e7: Pull complete
    e6f7c7e5c03e: Pull complete
    Digest: sha256:147913621d9cdea08853f6ba9116c2e27a3ceffecf3b492983ae97c3d643fbbe
    Status: Downloaded newer image for ubuntu:16.04
    
上面的命令中没有给出 Docker 镜像仓库地址，因此将会从 Docker Hub 获取镜像。而镜像名称是 ubuntu:16.04，因此将会获取官方镜像 library/ubuntu 仓库中标签为 16.04 的镜像。

从下载过程中可以看到我们之前提及的分层存储的概念，镜像是由多层存储所构成。下载也是一层层的去下载，并非单一文件。下载过程中给出了每一层的 ID 的前 12 位。并且下载结束后，给出该镜像完整的 sha256 的摘要，以确保下载一致性。

在使用上面命令的时候，你可能会发现，你所看到的层 ID 以及 sha256 的摘要和这里的不一样。这是因为官方镜像是一直在维护的，有任何新的 bug，或者版本更新，都会进行修复再以原来的标签发布，这样可以确保任何使用这个标签的用户可以获得更安全、更稳定的镜像。

从镜像启动并运行容器:
```bash
$ docker run -it --rm -p 8080:80 -v ~/index.html:/var/www/html/index.html ubuntu:16.04 bash
root@467edd01c65b:/#
```

docker run 就是运行容器的命令，下面简要的说明一下上面用到的参数。

+ -it：这是两个参数，一个是 -i：交互式操作，一个是 -t 终端。我们这里打算进入 bash 执行一些命令并查看返回结果，因此我们需要交互式终端。
+ --rm：这个参数是说容器退出后随之将其删除。默认情况下，为了排障需求，退出的容器并不会立即删除，除非手动 docker rm。我们这里只是随便执行个命令，看看结果，不需要排障和保留结果，因此使用 --rm 可以避免浪费空间。
+ -p: 将容器内端口(80)映射到宿主机端口(8080)
+ -v: 把宿主机上的目录(文件 ~/index.html)挂载到容器里（/var/www/html/index.html）
+ ubuntu:16.04：这是指用 ubuntu:16.04 镜像为基础来启动容器。
+ bash：放在镜像名后的是命令，这里我们希望有个交互式 Shell，因此用的是 bash。

下面安装并启动nginx：
```bash
$ echo "<h1>hello</h1>" > ~/index.html
$ docker run -it -p 8080:80 -v ~/index.html:/var/www/html/index.html ubuntu:16.04 bash
root@467edd01c65b:/# apt-get update
root@467edd01c65b:/# apt-get install nginx
root@467edd01c65b:/# /etc/init.d/nginx start
```
访问localhost:8080 便可看到之前挂载进容器的index.html页面

一些常用操作:
```bash
root@467edd01c65b exit              //退出交互式Shell
$ docker ps                         //列出当前运行的容器(-a 显示所有)
$ docker exec -it 467edd01c65b bash //进入容器
$ docker stop 467edd01c65b          //停止容器(开启用start)
$ docker commit 467edd01c65b test_docker:1.1 //从一个容器的修改中创建一个新的镜像；
$ docker images                     //列出存在的镜像
$ docker save test_docker:1.1 > test_docker_1.tar //导出镜像
$ docker load  < test_docker_1.tar  //导入镜像存储文件到本地镜像库（save的逆操作）
$ docker export 467edd01c65b > ubuntu16.04.tar  //导出容器快照(将丢弃所有的历史记录和元数据信息,即仅保存容器当时的快照状态)
$ cat ./ubuntu16.04.tar | docker import - ubuntu_1:16.04 //用 ./ubuntu16.04.tar 文件创建 ubuntu_1:16.04 的镜像(export的逆操作)
$ docker rm  467edd01c65b           //删除给定的若干个容器；
$ docker rmi 467edd01c65b           //删除给定的若干个镜像；
```

关于使用 docker commit 定制镜像：

    使用 docker commit 命令虽然可以比较直观的帮助理解镜像分层存储的概念，但是实际环境中并不会这样使用。
    
    例如安装软件包、编译构建，那会有大量的无关内容被添加进来，如果不小心清理，将会导致镜像极为臃肿。
    
    此外，使用 docker commit 意味着所有对镜像的操作都是黑箱操作，生成的镜像也被称为黑箱镜像，换句话说，就是除了制作镜像的人知道执行过什么命令、怎么生成的镜像，别人根本无从得知。而且，即使是这个制作镜像的人，过一段时间后也无法记清具体在操作的。虽然 docker diff 或许可以告诉得到一些线索，但是远远不到可以确保生成一致镜像的地步。这种黑箱镜像的维护工作是非常痛苦的。
    
    而且，回顾之前提及的镜像所使用的分层存储的概念，除当前层外，之前的每一层都是不会发生改变的，换句话说，任何修改的结果仅仅是在当前层进行标记、添加、修改，而不会改动上一层。如果使用 docker commit 制作镜像，以及后期修改的话，每一次修改都会让镜像更加臃肿一次，所删除的上一层的东西并不会丢失，会一直如影随形的跟着这个镜像，即使根本无法访问到。这会让镜像更加臃肿。

关于数据卷：

    数据卷 是一个可供一个或多个容器使用的特殊目录，它绕过 UFS，可以提供很多有用的特性：
    
        - 数据卷可以在容器之间共享和重用
        - 对数据卷的修改会立马生效
        - 对数据卷的更新，不会影响镜像
        - 数据卷默认会一直存在，即使容器被删除

> 注意：数据卷 的使用，类似于 Linux 下对目录或文件进行 mount，镜像中的被指定为挂载点的目录中的文件会隐藏掉，能显示看的是挂载的 数据卷。

创建一个数据卷
```bash
$ docker volume create my-vol
```

关于容器互联:
通过将容器加入自定义的 Docker 网络来连接多个容器.
下面先创建一个新的 Docker 网络:
```bash
$ docker network create -d bridge my-net
```
-d 参数指定 Docker 网络类型，有 bridge overlay。其中 overlay 网络类型用于 Swarm mode。

运行一个容器并连接到新建的 my-net 网络
```bash
$ docker run -it --rm --name busybox1 --network my-net busybox sh
```
打开新的终端，再运行一个容器并加入到 my-net 网络
```bash
$ docker run -it --rm --name busybox2 --network my-net busybox sh
```
下面通过 ping 来证明 busybox1 容器和 busybox2 容器建立了互联关系。
在 busybox1 容器输入以下命令

    / # ping busybox2
    PING busybox2 (172.19.0.3): 56 data bytes
    64 bytes from 172.19.0.3: seq=0 ttl=64 time=0.072 ms
    64 bytes from 172.19.0.3: seq=1 ttl=64 time=0.118 ms

#### 3. 使用 Dockerfile 定制镜像
    从 docker commit 可以了解到，镜像的定制实际上就是定制每一层所添加的配置、文件。如果我们可以把每一层修改、安装、构建、操作的命令都写入一个脚本，用这个脚本来构建、定制镜像，那么之前提及的无法重复的问题、镜像构建透明性的问题、体积的问题就都会解决。这个脚本就是 Dockerfile。
    
    Dockerfile 是一个文本文件，其内包含了一条条的指令(Instruction)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

具体使用:

- todo...

<br>
#### 4. 使用 docker-compose 快速编排容器集群
    根据Dockerfile最佳实践，应该保证在一个容器中只运行一个进程。将多个应用解耦到不同容器中，保证了容器的横向扩展和复用。例如 web 应用应该包含三个容器：web应用、数据库、缓存。
    
    docker-compose 允许用户通过一个单独的 docker-compose.yml 模板文件（YAML 格式）来定义一组相关联的应用容器为一个项目（project）。

    Compose 中有两个重要的概念：
        - 服务 (service)：一个应用的容器，实际上可以包括若干运行相同镜像的容器实例。
        - 项目 (project)：由一组关联的应用容器组成的一个完整业务单元，在 docker-compose.yml 文件中定义。

安装：(通过直接下载编译好的编译好的二进制文件)
```bash
sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Compose 模板文件
默认的模板文件名称为 docker-compose.yml，格式为 YAML 格式。

    version: "3"
    
    services:
      webapp:
        image: examples/web
        ports:
          - "80:80"
        volumes:
          - "/data"

注意每个服务都必须通过 image 指令指定镜像或 build 指令（需要 Dockerfile）等来自动构建生成镜像。

如果使用 build 指令，在 Dockerfile 中设置的选项(例如：CMD, EXPOSE, VOLUME, ENV 等) 将会自动被获取，无需在 docker-compose.yml 中再次设置。

实战 - 通过 [laradock](https://github.com/laradock/laradock) 快速构建 lnmp 环境:

- 克隆 laradock 

```bash
git clone https://github.com/Laradock/laradock.git
```

- 创建 .env 环境变量文件并做一些基本配置 

```bash
cd laradock
cp env-example .env
vim .env
```

    APP_CODE_PATH_HOST=../nines         //web应用路径
    MYSQL_DATABASE=wcwx
    MYSQL_ROOT_PASSWORD=roottoor
    ...


- 配置php项目数据库主机
```php
'hostname' => 'mysql'                      //thinkphp(database.php)
```

- 根据所用框架配置 nginx 
```bash
vim nginx/sites/default.conf
```

    server {
        ...
        root /var/www;                                          #项目根目录
    
        location / {
            if ( !-e $request_filename) {
                rewrite ^(.*)$ /index.php?s=/$1 last;       #url重写规则
                break;
            }  
        }
    
    }
    
- 设置mysql初始化数据
将sql文件放入 ./mysql/docker-entrypoint-initdb.d 目录并在sql中设置mysql用户权限
```sql
GRANT ALL ON `wcwx`.* TO 'default'@'%' ;
FLUSH PRIVILEGES ;
```

- 挂载目录到容器
```bash
$ vim docker-compose.yml
```

    php-fpm:
        ...
        volumes:
            - /vmstemplate:/vmstemplate
- 构建并运行
```bash
docker-compose up -d nginx mysql workspace 
```





