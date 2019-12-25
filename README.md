# Re-docker
重学Docker  进阶K8S

### 镜像构建上下文（Context）
这就引入了上下文的概念。当构建的时候，用户会指定构建镜像上下文的路径，docker build 命令得知这个路径后，会将路径下的所有内容打包，然后上传给 Docker 引擎。这样 Docker 引擎收到这个上下文包后，展开就会获得构建镜像所需的一切文件。

如果在 Dockerfile 中这么写：
```
COPY ./package.json /app/
```

这并不是要复制执行 docker build 命令所在的目录下的 package.json，也不是复制 Dockerfile 所在目录下的 package.json，而是复制 上下文（context） 目录下的 package.json。

因此，COPY 这类指令中的源文件的路径都是相对路径。这也是初学者经常会问的为什么 COPY ../package.json /app 或者 COPY /opt/xxxx /app 无法工作的原因，因为这些路径已经超出了上下文的范围，Docker 引擎无法获得这些位置的文件。如果真的需要那些文件，应该将它们复制到上下文目录中去。

现在就可以理解刚才的命令 docker build -t nginx:v3 . 中的这个 .，实际上是在指定上下文的目录，docker build 命令会将该目录下的内容打包交给 Docker 引擎以帮助构建镜像。

如果观察 docker build 输出，我们其实已经看到了这个发送上下文的过程：

(docker build 的时候会把指定目录全部提交  ，其中的资源操控请使用相对路径)

## 命令大全

### COPY
	- `COPY [--chown=<user>:<groip>] <源文件>...<目标文件>`
	- `COPY package.json /usr/src/app/`
	- `COPY hom* /mydir/`  可以加通配符   匹配规则filepath.Match
	- `COPY hom?.txt /mydir/` 
源文件的各种元数据都会保留。比如读、写、执行权限、文件变更时间等。这个特性对于镜像定制很有用。特别是构建相关文件都在使用 Git 进行管理的时候。

在使用该指令的时候还可以加上 --chown=<user>:<group> 选项来改变文件的所属用户及所属组。
```
COPY --chown=55:mygroup files* /mydir/
COPY --chown=bin files* /mydir/
COPY --chown=1 files* /mydir/
COPY --chown=10:11 files* /mydir/
```	

### ADD 高级复制
COPY PRO
ADD 指令和 COPY 的格式和性质基本一致。但是在 COPY 基础上增加了一些功能。

比如 <源路径> 可以是一个 URL，这种情况下，Docker 引擎会试图去下载这个链接的文件放到 <目标路径> 去。下载后的文件权限自动设置为 600，如果这并不是想要的权限，那么还需要增加额外的一层 RUN 进行权限调整，另外，如果下载的是个压缩包，需要解压缩，也一样还需要额外的一层 RUN 指令进行解压缩。所以不如直接使用 RUN 指令，然后使用 wget 或者 curl 工具下载，处理权限、解压缩、然后清理无用文件更合理。因此，这个功能其实并不实用，而且不推荐使用。

如果 <源路径> 为一个 tar 压缩文件的话，压缩格式为 gzip, bzip2 以及 xz 的情况下，ADD 指令将会自动解压缩这个压缩文件到 <目标路径> 去。

### CMD容器启动命令
CMD 指令的格式和RUN相识
	- shell格式: `CMD <命令>`
	- exec格式: `CMD ["可执行文件","参数1","参数2" ...]`
	- 参数列表格式：CMD ["参数1", "参数2"...]。在指定了 ENTRYPOINT 指令后，用 CMD 指定具体的参数。
Docker 不是虚拟机，容器就是进程。既然是进程，那么在启动容器的时候，需要指定所运行的程序及参数。CMD 指令就是用于指定默认的容器主进程的启动命令的。

MD 就不得不提容器中应用在前台执行和后台执行的问题。这是初学者常出现的一个混淆。

Docker 不是虚拟机，容器中的应用都应该以前台执行，而不是像虚拟机、物理机里面那样，用 systemd 去启动后台服务，容器内没有后台服务的概念。

一些初学者将 CMD 写为：
```
CMD service nginx start
```
然后发现容器执行后就立即退出了。甚至在容器内去使用 systemctl 命令结果却发现根本执行不了。这就是因为没有搞明白前台、后台的概念，没有区分容器和虚拟机的差异，依旧在以传统虚拟机的角度去理解容器。

对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出，其它辅助进程不是它需要关心的东西。

而使用 service nginx start 命令，则是希望 upstart 来以后台守护进程形式启动 nginx 服务。而刚才说了 CMD service nginx start 会被理解为 CMD [ "sh", "-c", "service nginx start"]，因此主进程实际上是 sh。那么当 service nginx start 命令结束后，sh 也就结束了，sh 作为主进程退出了，自然就会令容器退出。

正确的做法是直接执行 nginx 可执行文件，并且要求以前台形式运行。比如：
```
CMD ["nginx", "-g", "daemon off;"]
```

### ENTRYPOINT 入口点
ENTRYPOINT 的格式和 RUN 指令格式一样，分为 exec 格式和 shell 格式。

ENTRYPOINT 的目的和 CMD 一样，都是在指定容器启动程序及参数。ENTRYPOINT 在运行时也可以替代，不过比 CMD 要略显繁琐，需要通过 docker run 的参数 --entrypoint 来指定。

当指定了 ENTRYPOINT 后，CMD 的含义就发生了改变，不再是直接的运行其命令，而是将 CMD 的内容作为参数传给 ENTRYPOINT 指令，换句话说实际执行时，将变为：

```
<ENTRYPOINT> "<CMD>"
```

### 使用场景1: 接受用户输入

```
FROM ubuntu:18.04
RUN apt-get update \
    && apt-get install -y curl \
    && rm -fr /var/lib/apt/lists/*
ENTRYPOINT ["curl","-s","https://ip.cn"]
```

```
dollarkiller@dollarkiller-virtual-machine:~/Github/Re-docker/day1/test2$ docker run hello:v1 
{"ip": "222.92.46.234", "country": "江苏省苏州市", "city": "电信"}
dollarkiller@dollarkiller-virtual-machine:~/Github/Re-docker/day1/test2$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
49525d8a6c72        mysql:5.7           "docker-entrypoint.s…"   4 weeks ago         Up 5 days           0.0.0.0:3306->3306/tcp, 33060/tcp   mysql
dollarkiller@dollarkiller-virtual-machine:~/Github/Re-docker/day1/test2$ docker ps -a
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS                     PORTS                               NAMES
7d3e8fe3c68b        hello:v1              "curl -s https://ip.…"   12 seconds ago      Exited (0) 9 seconds ago                                       reverent_lederberg
a39107b9df73        nginx:1.17.5-alpine   "nginx -g 'daemon of…"   4 weeks ago         Exited (0) 4 weeks ago                                         test
49525d8a6c72        mysql:5.7             "docker-entrypoint.s…"   4 weeks ago         Up 5 days                  0.0.0.0:3306->3306/tcp, 33060/tcp   mysql
dollarkiller@dollarkiller-virtual-machine:~/Github/Re-docker/day1/test2$ docker rm reverent_lederberg 
reverent_lederberg
dollarkiller@dollarkiller-virtual-machine:~/Github/Re-docker/day1/test2$ docker run hello:v1 -i
HTTP/2 200 
date: Wed, 25 Dec 2019 10:24:26 GMT
content-type: application/json; charset=UTF-8
set-cookie: __cfduid=de698eb3d94964db2245b2f823be192901577269466; expires=Fri, 24-Jan-20 10:24:26 GMT; path=/; domain=.ip.cn; HttpOnly; SameSite=Lax
cf-cache-status: DYNAMIC
expect-ct: max-age=604800, report-uri="https://report-uri.cloudflare.com/cdn-cgi/beacon/expect-ct"
server: cloudflare
cf-ray: 54aa1af27f247758-LAX

{"ip": "222.92.46.234", "country": "江苏省苏州市", "city": "电信"}
dollarkiller@dollarkiller-virtual-machine:~/Github/Re-docker/day1/test2$ 
```

EBTRYPOIN 会接受用户的输入 放到执行命令的后端

### 使用场景2： 应用运行前的准备工作

启动容器就是启动主进程，但有些时候，启动主进程前，需要一些准备工作。

比如 mysql 类的数据库，可能需要一些数据库配置、初始化的工作，这些工作要在最终的 mysql 服务器运行之前解决。

此外，可能希望避免使用 root 用户去启动服务，从而提高安全性，而在启动服务前还需要以 root 身份执行一些必要的准备工作，最后切换到服务用户身份启动服务。或者除了服务外，其它命令依旧可以使用 root 身份执行，方便调试等。

这些准备工作是和容器 CMD 无关的，无论 CMD 为什么，都需要事先进行一个预处理的工作。这种情况下，可以写一个脚本，然后放入 ENTRYPOINT 中去执行，而这个脚本会将接到的参数（也就是 <CMD>）作为命令，在脚本最后执行。比如官方镜像 redis 中就是这么做的：

```
FROM alpine:3.4
...
RUN addgroup -S redis && adduser -S -G redis redis
...
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 6379
CMD [ "redis-server" ]
```

可以看到其中为了 redis 服务创建了 redis 用户，并在最后指定了 ENTRYPOINT 为 docker-entrypoint.sh 脚本。

```
#!/bin/sh
...
# allow the container to be started with `--user`
if [ "$1" = 'redis-server' -a "$(id -u)" = '0' ]; then
    chown -R redis .
    exec su-exec redis "$0" "$@"
fi

exec "$@"
```

该脚本的内容就是根据 CMD 的内容来判断，如果是 redis-server 的话，则切换到 redis 用户身份启动服务器，否则依旧使用 root 身份执行。比如：

### ENV 设置环境变量

