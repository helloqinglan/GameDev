## 创建镜像的两种方法：
- docker commit
- Dockerfile和docker build

## CMD和ENTRYPOINT的区别
这两个命令都指定在了镜像中要运行的命令，但有一个重要的区别：
- CMD 在docker run不带参数时将简单的执行CMD指定的命令
- ENTRYPOINT 使得镜像的行为更像是一个二进制文件

### 详细描述：
- 如果Dockerfile中只使用CMD，那么在docker run命令不带参数的时候，CMD指定的命令会被执行
- 如果Dockerfile中只使用ENTRYPOINT，docker run的参数会被传递给entrypoint作为参数，或者当docker run不带参数时直接执行entrypoint
- 如果Dockerfile中同时使用了CMD和ENTRYPOINT，并且docker run不带参数，那么CMD后的参数将会被传递给entrypoint

在使用ENTRYPOINT时要格外小心，它会使得在镜像内部获得shell接口变得很困难。如果你的镜像被设计用来执行一个很简单的命令，那这不会是问题，但是如果你希望镜像被这样使用则会迷惑用户：
```
# docker run -i -t <your image> /bin/bash
```

镜像的entrypoint也能使用docker run的--entrypoint参数来覆盖。如果你使用了entrypoint，可以告诉你的用户用这种方法来获得shell：
```
# docker run -i --entrypoint /bin/bash -t <your image>
```

## CMD和ENTRYPOING的Array和String形式的区别
- 类似CMD [ "LS", "/" ]这样的array参数会直接执行指定的命令及参数
- 类似CMD LS /这样的string参数会增加/bin/sh -c前缀
建议使用array形式的命令参数

## 在Wrapper Scripts中启动应用进程时始终使用exec命令
很多人会使用wrapper scripts来做环境初始化的工作，然后在最后启动真正的应用进程。记住一定要使用exec来执行应用进程，以确保script进程被应用进程替换掉。

如果不使用exec命令，docker发送到容器内的信号将到到达wrapper script进程，而不是我们所期望的应用进程。

比如使用CTRL+C命令来杀掉容器进程时，如果是使用exec来启动的服务进程，docker会把SIGINT信息发送到服务进程，进程会被结束。如果没有使用exec，docker会把SIGINT信息发到wrapper script，这样服务进程将不会被结束。

## 始终用EXPOSE暴露重要的端口
EXPOSE指令让容器的端口在主机系统和其他容器内可见。虽然也可以在docker run时指定暴露的端口，但是在Dockerfile中使用EXPOSE指令具有更直观的好处：
- 导出的端口会在docker ps命令中显示
- 导出的端口也会在docker inspect命令中显示
- 导出的端口在使用link时会连接到其他容器内

## 适当的使用卷 (Volumes)
VOLUME指令或者docker run -v参数告诉docker将文件存储到宿主机的目录上，而不是默认的容器内文件系统。使用Volumes能够带来一系列好处：
- 使用--volumes-from能够在容器间共享卷
- 读写大文件时效率更高

与容器内部文件系统的区别：
- docker commit不会包含外部卷的内容
- Volumes使用引用计数来管理其生命周期, 直到没有任何容器引用该卷时才会被销毁

## 使用USER命令
默认情况下docker以root用户来运行容器实例，也就意味着进程拥有对整个容器的完全控制权限。更高的权限意味着更多的安全风险，所以尽可能的用USER命令来指定非root用户。

如果你的应用没有创建自己的用户，可以在Dockerfile中创建一个用户和组：
```
RUN groupadd -r swuser -g 433 &&\
useradd -u 431 -r -g swuser -d <homedir> -s /sbin/nologin -c "Docker image user" swuser && \
chown -R swuser:swuser <homedir>
```

当指定其他基础镜像时要注意，Dockerfile的默认用户是父镜像的用户，比如说你的镜像从修改了用户为swuser的镜像继承而来，那么你的Dockerfile中的RUN命令也会使用swuser用户

如果希望更改当前用户，可以使用USER指令来更换
```
USER root
RUN yum install -y <some package>
USER swuser
```

原始链接：
http://www.projectatomic.io/docs/docker-image-author-guidance/