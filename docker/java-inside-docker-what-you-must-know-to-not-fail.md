Docker的资源限制参数 -m -memory -memory-swap
Kubernetes的资源限制参数 -limits
如果进程超过了资源限制，Linux内核会杀掉该进程
但是JVM并不清楚这些限制值

示例：
```
# docker run -it --rm --name mycontainer150 -p 8080:8080 -m 150m rafabene/java-container:openjdk
```

使用"/api/memeory"接口来获取内存操作：
```
# curl http://`docker-machine ip docker1024`:8080/api/memory
```

结果如下：
```
"Allocated more than 80% (219.8MiB) of the max allowed JVM memory size (241.7MiB)"
```

这里有两个问题：
- 为什么JVM允许的最大内存是241.7MiB？
- 容器限制的内存大小为150MB，为什么会允许JVM申请220MB内存？

首先，从JVM文档的描述，"maximum heap size"是1/4的物理内存，我们创建的这个Docker虚拟机内存为1G，所以允许的堆内存最大值约为260MB
从进程启动时的输出信息能够看到
```
$docker logs mycontainer150 | grep -i MaxHeapSize
uintx MaxHeapSize := 262144000 {product}
```

其次，我们需要理解docker的启动参数 -m 150M的含义，docker守护进程会限制150M的RAM空间和150M的Swap空间。这样，进程实际上能够分配300M内存，这也就解释了-m 150M时进程分配了241M内存却还没有被杀掉 (OOM)

修改一下虚拟机的内存大小为8G再试一次
```
docker-machine create -d virtualbox -virtualbox-memory '8192' docker8192
```

并且我们增加内存大小，限制进程可用内存为600M
```
docker run -it --name mycontainer -p 8080:8080 -m 600M rafabene/java-container:openjdk
```

此时MaxHeapSize将为2G (8G的1/4)
```
docker logs mycontainer | grep -i MaxHeapSize
uintx MapHeapSize := 2092957696
```

而进程可以分配的内存大小只有1.2G (600M RAM + 600M Swap)，此时将会发生OOM，进程被杀掉

很显然，通过增加内存来让JVM自己设置参数的方法有时会带来问题
我们应该使用-Xmx参数来自己设置进程可以使用的最大堆内存

在Dockerfile中添加环境变量的使用
```
CMD java -XX:+PrintFlagsFinal $JAVA_OPTIONS -jar java-container.jar
```

```
docker run -d --name mycontainer8g -p 8080:8080 -m 600M -e JAVA_OPTIONS='-Xmx300m' rafabene/java-container:openjdk

docker logs mycontainer8g | grep -i MaxHeapSize
uintx MaxHeapSize := 814572800 {product}
```

在Kubernetes上则通过-env-[key=value]的方式来设置环境变量
```
kubectl run mycontainer --image=rafabene/java-container:openjdk-evn --limits='memory=600Mi' --env="JAVA_OPTIONS=-Xmx300m"

kubectl logs $(kubectl get pods | grep mycontainer | awk '{print $1}' | had -1 | grep MaxHeapSize)
uintx MaxHeapSize := 314572800 {product*}
```