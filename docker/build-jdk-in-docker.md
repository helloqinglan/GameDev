


```
FROM ubuntu:18.04

RUN apt-get update && \
    apt-get install -y -q \
    mercurial && \
    rm -rf /var/lib/apt/lists/*

VOLUME /mnt
WORKDIR /mnt

ENTRYPOINT [ "hg" ]
```

```
docker run -v ~/openjdk8u-hg:/mnt hgclient:v1 clone https://hg.openjdk.java.net/jdk8u/jdk8u/ /mnt
docker run -v ~/openjdk8u-hg:/mnt -it --entrypoint /bin/sh hgclient:v1 -c /mnt/get_source.sh
```



```
FROM ubuntu:18.04

RUN apt-get update && \
    apt-get install -y -q \
    build-essential gawk m4 openjdk-8-jdk ibasound2-dev libcups2-dev libxrender-dev \
    xorg-dev xutils-dev x11proto-print-dev binutils ibmotif-common libxm4 libuil4 \
    libmrm4 ibmagic-mgc libmagic1 zip ant && \
    rm -rf /var/lib/apt/lists/*

VOLUME /mnt
WORKDIR /mnt

ENTRYPOINT [ "/bin/sh" ]
```


```
docker run -v ~/openjdk8u-hg:/mnt -it jdkbuilder:v1 /bin/sh
bash configure
make images
```

[编译指南 https://hg.openjdk.java.net/jdk/jdk/raw-file/tip/doc/building.html](https://hg.openjdk.java.net/jdk/jdk/raw-file/tip/doc/building.html)