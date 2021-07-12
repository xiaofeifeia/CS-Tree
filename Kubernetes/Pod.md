### Pod 的概念

**Pod 是 Kubernetes 项目中最小的 API 对象，也即 Kubernetes 项目的原子调度单位。**

假设有如下场景：

> Linux 系统中负责日志处理的 rsyslogd 由三个进程组成：imklog 模块、imuxsock 模块、rsyslogd 自己的 main 函数主进程。这三个进程一定要运行在同一台机器上，否则，它们之间基于 Socket 的通信和文件交换，都会出现问题。
>
> 现在要把 rsyslogd 这个应用容器化，由于受限于容器的“单进程模型”（容器没有管理多个进程的能力），这三个模块必须分别被制作成三个不同的容器。而在这三个容器运行的时候，它们设置的内存配额都是 1 GB。
>
> 假设 Kubernetes 集群上有两个节点：node-1 有 3 GB 可用内存、node-2 有 2.5 GB 可用内存，此时如果使用 Docker Swarm 来运行这个 rsyslogd 程序，为了能够让这三台机器都运行在同一台机器上，必须要为另外两个容器设置一个 `affinity = main` 的约束，而且要顺序的启动三个容器，当 main 容器和 imklog 容器都被调度到了 node-2 节点上时，此时可用内存仅剩 0.5 GB 了，并不足以运行 imuxsocket 容器，而根据 `affinity = main` 约束， imuxsocket 容器又只能运行在 node-2 节点上，于是就出现了冲突。

而在 Kubernetes 中，Pod 是原子调度单位，这就意味着，Kubernetes 项目的调度器，是同一按照 Pod 而非容器的资源需求进行计算的。所以采用 Kubernetes 调度以上容器时，只会去考虑可用内存为 3GB 的 node-1 节点进行绑定，而根本不会考虑 node-2 节点。

这样的容器间的紧密协作，称为“超亲密关系”。包括但不限于：互相之间会发生直接的文件交换、使用 localhost 或者 Socket 文件进行本地通信、非常频繁的远程调用、需要共享某些 Linux Namespace（比如 Network Namespace）等等。优先考虑创建在一个 Pod 中。

**Pod 只是一个逻辑概念，它其实是一组共享了某些资源的容器。Pod 里的所有容器，共享的是同一个 Network Namespace，并且可以声明共享同一个 Volume。**

假设有如下场景：

> 有 A、B 两个容器，容器 A 需要共享容器 B 的网络和 Volume。
>
> 很容易想到通过 `docker run --net=B --volumes-from=B --name=A image-A ...` 命令就能实现。但这样存在一个问题：容器 B 必须比容器 A 先启动，这就意味着容器 A 和 容器 B 之间不再是对等关系，而是拓扑关系。

在 Kubernetes 中， Pod 的实现需要使用一个中间容器，即**镜像名为“k8s.gcr.io/pause”、镜像采用汇编语言编写、永远处于“暂停”状态、镜像解压后大小只有 100 - 200 KB 左右、占用资源极少** 的 Infra 容器。在 Pod 中，Infra 容器永远都是第一个被创建的容器，其他容器通过 `Join Network Namespace` 的方式与 Infra 容器关联在一起，如下图：

![img](assets/8c016391b4b17923f38547c498e434cf.png)

在 Infra 容器 Hold 住 Network Namespace 之后，用户容器就可以加入到 Infra 容器的 Network Namespace 中了，这些容器在宿主机上的 Namespace 指向是完全一样的。这也就意味着：

1. 容器 A 和容器 B 可以直接使用 localhost 进行通信
2. 容器 A 和容器 B 看到的网络设备跟 Infra 容器看到的完全一样
3. 一个 Pod 只有一个 IP 地址，也就是这个 Pod 的 Network Namespace 对应的 IP 地址
4. 其他网络资源，都是一个 Pod 一份，并且被该 Pod 中的所有容器共享
5. Pod 的生命周期只跟 Infra 容器一致，而与容器 A 和 B 无关
6. 同一个 Pod 里面的所有用户容器的进出流量，可以认为都是通过 Infra 容器完成的
7. Kubernetes 的网络插件，应该考虑的是如何配置 Pod 的 Network Namespace，而不用关心用户容器。

因此，在 Kubernetes 项目中，只要把所有的 Volume 都定义在 Pod 层级，之后 Pod 内的容器只要声明挂载这个 Volume，就一定可以共享这个 Volume  对应的宿主机目录。

Pod 这种“超亲密关系”容器的设计思想，就是希望当用户想在一个容器里跑多个功能并不相关的应用时，应该优先考虑它们是不是更应该被描述成一个 Pod 里的多个容器。

> Pod，实际上是在扮演传统基础设施里“虚拟机”的角色；而容器，则是这个虚拟机里运行的用户程序。这样的设计，是为了使用户从传统环境（虚拟机环境）向 Kubernetes（容器环境）的迁移，更加平滑。

**所以 Pod 也可以看成是传统环境里的“机器”、而容器则是运行在这个“机器”里的“用户程序”。** 所以凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的。

### Pod 的使用

#### Pod 中几个重要字段的含义和用法：

**NodeSelector**：是一个供用户将 Pod 与 Node 进行绑定的字段，用法如下：

``` yaml
apiVersion: v1
kind: Pod
...
spec:
 nodeSelector:
   # 表示这个 Pod 只能运行在携带了“disktype: ssd”标签的节点上；否则将调度失败。
   disktype: ssd
```

**NodeName**：一旦 Pod 的这个字段被赋值，Kubernetes 项目就会被认为这个 Pod 已经经过了调度，调度的结果就是赋值的节点名字。所以，这个字段一般由调度器负责设置，但用户也可以设置它来“骗过”调度器，这个做法一般是在测试或者调试的时候才会用到。

**HostAliases**：定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容，在 Kubernetes 项目中，如果要设置 hosts 文件里的内容，一定要通过这种方法。否则如果直接修改了 hosts 文件的话，在 Pod 被删除重建之后，kubelet 会自动覆盖掉被修改的内容。用法如下：

``` yaml
apiVersion: v1
kind: Pod
...
spec:
  hostAliases:
  - ip: "10.1.2.3"
    hostnames:
    - "foo.remote"
    - "bar.remote"
...
```

这个 Pod 启动后，`/etc/hosts` 文件的内容将如下所示：

```shell
# Kubernetes-managed hosts file.
127.0.0.1 localhost
...
10.244.135.10 hostaliases-pod
10.1.2.3 foo.remote
10.1.2.3 bar.remote
```

**ShareProcessNamespace**：一旦 Pod 的这个字段被赋值为 True，就表示Pod 里的容器要共享 PID Namespace，用法如下：

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  shareProcessNamespace: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

创建 Pod，并连接到 shell 容器的 tty 上，执行 ps ax 命令，可以看到 nginx 容器的进程和 infra 容器的 /pause 进程：

``` shell
$ kubectl create -f nginx.yaml
$ kubectl attach -it nginx -c shell
/ # ps ax
PID   USER     TIME  COMMAND
    1 root      0:00 /pause
    8 root      0:00 nginx: master process nginx -g daemon off;
   14 101       0:00 nginx: worker process
   15 root      0:00 sh
   21 root      0:00 ps ax
```

类似的，定义了共享宿主机的 Network、IPC、 PID Namespace。就意味着 Pod 里的所有容器，会直接使用宿主机的网络、直接与宿主机进行 IPC 通信、看到宿主机里正在运行的所有进程：

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  hostNetwork: true
  hostIPC: true
  hostPID: true
  containers:
  - name: nginx
    image: nginx
  - name: shell
    image: busybox
    stdin: true
    tty: true
```

**ImagePullPolicy**：定义了镜像拉取的策略。值默认是 Always，即每次创建 Pod 都重新拉取一次镜像。另外，当容器的镜像是类似于 nginx 或者 nginx:latest 这样的名字时，ImagePullPolicy 也会被认为 Always。而如果它的值被定义为 Never 或者 IfNotPresent，则意味着 Pod 永远不会主动拉取这个镜像，或者只在宿主机上不存在这个镜像时才拉取。

**Lifecycle**：定义的是 Container Lifecycle Hooks，是在容器状态发生变化时触发的一系列“钩子”：

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-demo
spec:
  containers:
  - name: lifecycle-demo-container
    image: nginx
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
```

**postStart** 是指在容器启动后，立刻执行一个指定的操作。虽然 postStart 定义的操作，是在 Docker 容器 ENTRYPOINT 执行之后，但它并不严格保证顺序。也就是说，在 postStart 启动时，ENTRYPOINT 有可能还没有结束。如果 postStart 执行超时或者错误，Kubernetes 会在该 Pod 的 Events 中报出该容器启动失败的错误信息，导致 Pod 也处于失败的状态。

**preStop** 是指在容器被杀死之前，执行一个指定的操作。preStop 操作的执行是同步的。所以，它会阻塞当前的容器杀死流程，直到这个 Hook 定义操作完成之后，才允许容器被杀死。

#### Pod 对象在 Kubernetes 中的生命周期

Pod 生命周期的变化主要体现在 Pod API 对象的 Status 部分，其中 pod.status.phase，就是 Pod 的当前状态：

1. Pending。这个状态意味着，Pod 的 YAML 文件已经提交给了 Kubernetes，API 对象已经被创建并保存在 Etcd 当中。但是，这个 Pod 里有些容器因为某种原因而不能被顺利创建。比如调度不成功。
2. Running。这个状态下，Pod 已经调度成功，跟一个具体的节点绑定。它包含的容器都已经创建成功，并且至少有一个正在运行中。
3. Succeeded。这个状态意味着，Pod 里的所有容器都正常运行完毕，并且已经退出了。这种情况在运行一次性任务时最为常见。
4. Failed。这个状态下，Pod 里至少有一个容器以不正常的状态退出。需要 Debug 这个容器的应用，比如查看 Pod 的 Events 和日志。
5. Unknown。这是一个异常状态，意味着 Pod 的状态不能持续地被 kubelet 汇报给 kube-apiserver，这很有可能是主从节点间的通信出现了问题。

Pod 对象的 Status 字段，还可以再细分出一组 Conditions。这些细分状态的值包括：PodScheduled、Ready、Initialized，以及 Unschedulable。它们主要用于描述造成当前 Status 的具体原因是什么。比如，Pod 当前的 Status 是 Pending，对应的 Condition 是 Unschedulable，这就意味着它的调度出现了问题。而其中，Ready 这个细分状态非常值得我们关注：它意味着 Pod 不仅已经正常启动（Running 状态），而且已经可以对外提供服务了。Pod 的这些状态信息，是判断应用运行情况的重要标准。

#### Projected Volume

Projected Volume 是 Kubernetes v1.11 之后的新特性，可以翻译为“投射数据卷”。在 Kubernetes 中，有几种特殊的 Volume 是为容器提供预先定义好的数据。从容器的角度来看，这些 Volume 里的信息就是仿佛是被 Kubernetes“投射”（Project）进入容器当中的。这正是 Projected Volume 的含义。

到目前为止，Kubernetes 支持的 Projected Volume 一共有四种：

1. Secret
2. ConfigMap
3. Downward API
4. ServiceAccountToken

**Secretd** 能把 Pod 想要访问的加密数据，存放到 Etcd 中。然后通过在 Pod 的容器里挂载 Volume 的方式，就可以访问到这些 Secret 里保存的信息了。Secret 最典型的使用场景，莫过于存放数据库的 Credential 信息：

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-projected-volume 
spec:
  containers:
  - name: test-secret-volume
    image: busybox
    args:
    - sleep
    - "86400"
    volumeMounts:
    - name: mysql-cred
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: mysql-cred
    projected:
      sources:
      - secret:
          name: user
      - secret:
          name: pass
```

这个 Pod 中定义了一个简单的容器。它声明挂载的 Volume 是 projected 类型。这个 Volume 的数据来源是名为 user 和 pass 的 Secret 对象，分别对应的是数据库的用户名和密码。它们是以 Secret 对象的方式交给 Kubernetes 保存的：

``` shell
$ cat ./username.txt
lollipop
$ cat ./password.txt
123456!

$ kubectl create secret generic user --from-file=./username.txt
$ kubectl create secret generic pass --from-file=./password.txt
```

查看这些 Secret 对象的只要执行一条 kubectl get 命令就可以了：

``` shell
$ kubectl get secrets
NAME           TYPE                                DATA      AGE
user          Opaque                                1         51s
pass          Opaque                                1         51s
```

也可以直接通过编写 YAML 文件的方式来创建这个 Secret 对象。通过编写 YAML 文件创建出来的 Secret 对象只有一个。但它的 data 字段，却以 Key-Value 的格式保存了两份 Secret 数据。Secret 对象要求这些数据必须是经过 Base64 转码的，在生产环境中，还需要在 Kubernetes 中开启 Secret 的加密插件，增强数据的安全性：

``` yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  user: YWRtaW4=
  pass: MWYyZDFlMmU2N2Rm
```

保存在 Etcd 里的用户名和密码信息，以文件的形式存在于容器的 Volume 目录里：

``` shell
$ kubectl create -f test-projected-volume.yaml
$ kubectl exec -it test-projected-volume -- /bin/sh
$ ls /projected-volume/
user
pass
$ cat /projected-volume/user
root
$ cat /projected-volume/pass
1f2d1e2e67df
```

这种通过挂载方式进入到容器里的 Secret，对应的 Etcd 里的数据被更新时，Volume 里的文件内容也会被更新，因为 kubelet 组件在定时维护这些 Volume。

**ConfigMap** 与 Secret 类似，用法几乎完全相同，主要区别在于 ConfigMap 保存的是不需要加密的、应用所需的配置信息。

将一个 Java 应用所需的配置文件（*.properties）保存在 ConfigMap 中：

``` shell
# .properties文件的内容
$ cat example/ui.properties
color.good=purple
color.bad=yellow
allow.textmode=true
how.nice.to.look=fairlyNice

# 从.properties文件创建ConfigMap
$ kubectl create configmap ui-config --from-file=example/ui.properties

# 查看这个ConfigMap里保存的信息(data)
$ kubectl get configmaps ui-config -o yaml
apiVersion: v1
data:
  ui.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
    how.nice.to.look=fairlyNice
kind: ConfigMap
metadata:
  name: ui-config
  ...
```

**Downward API** 能让 Pod 里的容器直接获取到这个 Pod API 对象本身的信息。Downward API 能够获取到的信息，一定是 Pod 里的容器进程启动之前就能够确定下来的信息。而如果想要获取 Pod 容器运行后才会出现的信息，比如进程 PID，则应该考虑在 Pod 里定义一个 sidecar 容器。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-downwardapi-volume
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
spec:
  containers:
    - name: client-container
      image: k8s.gcr.io/busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/podinfo/labels ]]; then
            echo -en '\n\n'; cat /etc/podinfo/labels; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc/podinfo
          readOnly: false
  volumes:
    - name: podinfo
      projected:
        sources:
        - downwardAPI:
            items:
              - path: "labels"
                fieldRef:
                  fieldPath: metadata.labels
```

上面这个 Pod 的 YAML 文件中，Volume 的数据来源，变成了 Downward API，它声明了要暴露 Pod 的 metadata.labels 信息给容器。通过这样的声明方式，当前 Pod 的 Labels 字段的值，就会被 Kubernetes 自动挂载成为容器里的 /etc/podinfo/labels 文件。而这个容器的启动命令，则是不断打印出 /etc/podinfo/labels 里的内容。当这个 Pod 创建之后，就可以通过 kubectl logs 指令，查看到这些 Labels 字段被打印出来：

``` shell
$ kubectl create -f dapi-volume.yaml
$ kubectl logs test-downwardapi-volume
cluster="test-cluster1"
rack="rack-22"
zone="us-est-coast"
```

**Service Account** 就是 Kubernetes 系统内置的一种“服务账户”，它是 Kubernetes 进行权限分配的对象。Service Account 的授权信息和文件，保存在它所绑定的一个特殊的 Secret 对象里的。这个特殊的 Secret 对象，就叫作 **ServiceAccountToken**。任何运行在 Kubernetes 集群上的应用，都必须使用这个 ServiceAccountToken 里保存的授权信息，才可以合法地访问 API Server。Kubernetes 有一个默认“服务账户”（default Service Account），每一个 Pod 都会自动声明一个类型是 Secret、名为 default-token-xxxx 的 Volume，然后自动挂载在每个容器的一个固定目录上。所以任何一个运行在 Kubernetes 里的 Pod，都可以直接使用，且无需显示地声明挂载它。

### 容器健康检查和恢复机制

在 Kubernetes 中，可以为 Pod 里的容器定义一个健康检查“探针”（Probe）。这样，kubelet 就会根据这个 Probe 的返回值决定这个容器的状态，而不是直接以容器镜像是否运行作为依据。这是生产环境中保证应用健康存活的重要手段：

``` yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: test-liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
```

在上面这个 Pod 中，容器 liveness 在启动之后做的第一件事，就是在 /tmp 目录下创建了一个 healthy 文件，以此作为自己已经正常运行的标志。30 s 后把这个文件删除掉。livenessProbe（健康检查）的类型是 exec，它会在容器启动后，在容器里面执行一条我们指定的命令，比如：“cat /tmp/healthy”。这时，如果这个文件存在，这条命令的返回值就是 0，Pod 就会认为这个容器不仅已经启动，而且是健康的。这个健康检查，在容器启动 5 s 后开始执行（initialDelaySeconds: 5），每 5 s 执行一次（periodSeconds: 5）。

Kubernetes 的 Pod 恢复机制，也叫 restartPolicy。它是 Pod 的 `pod.spec.restartPolicy` 字段，默认值是 Always：任何时候容器发生了异常，一定会被重建。

Pod 的恢复过程，永远都是发生在当前节点上，一旦一个 Pod 与一个节点（Node）绑定，除非 `pod.spec.node` 被修改，否则它永远都不会离开这个节点。

