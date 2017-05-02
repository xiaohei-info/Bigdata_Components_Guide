## 一、在生产环境中使用Pod来工作

本节将介绍一些在生产环境中运行应用非常有用的功能。

### 1、持久化存储

容器的文件系统只有当容器正常运行时有效，一旦容器奔溃或者重启，所有对文件系统的修改将会丢失，从一个原始的文件系统重新开始。   
所以为了实现更多的持久化信息，在文件系统之外你需要一个volume，volume对有状态的应用来说是非常重要的，例如键值对存储和数据库等。

例如，Redis是一个键值对的存储库，在[guestbook](https://github.com/kubernetes/kubernetes/blob/master/examples/guestbook)这个例子中使用过。   
我们可以通过一下的配置添加一个volume来存储需要持久化的数据：

```shell
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis
spec:
  template:
    metadata:
      labels:
        app: redis
        tier: backend
    spec:
      # Provision a fresh volume for the pod
      volumes:
        - name: data
          emptyDir: {}
      containers:
      - name: redis
        image: kubernetes/redis:v1
        ports:
        - containerPort: 6379
        # Mount the volume into the pod
        volumeMounts:
        - mountPath: /redis-master-data
          name: data   # must match the name of the volume, above
```

emptyDir这列的生命周期是随着pod的生命周期的，比任何一个容器的生命周期都要长，所以当容器奔溃或者重启的时候，这些持久化数据依然存在。

除了emptyDir提供的本地磁盘存储之外，k8s还提供了很多不同的网络存储方案，包括GCE的PD和EC2的EBS，这些是存储关键数据的首先方案并且会自动处理一些细节，例如在节点上mount和unmout设备。   

从这里看配置volume的全部信息：   
[the volumes doc](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/volumes.md) 

### 2、分配证书

为了和其他的应用、数据和和服务器进行互相验证，很多应用都需要证书，例如密码、OAuth tokens和TLS keys等。   
将这些证书存储在容器的镜像或者环境变量中是不理想的，因为这些证书可以被任何容器从镜像、配置文件、主机文件系统或者主机的Docker守护进程中复制出来。

为了方便容器之间交互敏感的证书，k8s提供了一个名为secrets的机制。   
Secret也是k8s中的一个资源，一组map类型的数据，例如，一个简单的包含用户名和密码的Secret可以这样定义：

```shell
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: dmFsdWUtMg0K
  username: dmFsdWUtMQ0K
```

和其他的资源一样，Secret也可以通过create命令来实例化，也可以通过get命令来查看：

```shell
$ kubectl create -f ./secret.yaml
secrets/mysecret
$ kubectl get secrets
NAME                  TYPE                                  DATA
default-token-v9pyz   kubernetes.io/service-account-token   2
mysecret              Opaque                                2
```

你需要在Pod或者Pod Template中引用这个Secret才能使用它，secret列使你可以在容器中将它像内存目录一样进行挂载。

```shell
apiVersion: v1
kind: ReplicationController
metadata:
  name: redis
spec:
  template:
    metadata:
      labels:
        app: redis
        tier: backend
    spec:
      volumes:
        - name: data
          emptyDir: {}
        - name: supersecret
          secret:
            secretName: mysecret
      containers:
      - name: redis
        image: kubernetes/redis:v1
        ports:
        - containerPort: 6379
        # Mount the volume into the pod
        volumeMounts:
        - mountPath: /redis-master-data
          name: data   # must match the name of the volume, above
        - mountPath: /var/run/secrets/super
          name: supersecret
```

点击这些查看更加详细的信息：   
[secrets document](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/secrets.md)   
[example](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/secrets)   
[design doc](https://github.com/kubernetes/kubernetes/blob/master/docs/design/secrets.md)

### 3、通过私有的镜像仓库进行校验

Secrets也可以用来通过镜像仓库的验证（[image registry credentials](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/images.md#using-a-private-registry)）。

首先，穿件一个.dockercfg文件，例如运行docker login <registry.domain>。
之后将.dockercfg的执行结果输入到一个Secret资源中，示例如下：

```shell
$ docker login
Username: janedoe
Password: ●●●●●●●●●●●
Email: jdoe@example.com
WARNING: login credentials saved in /Users/jdoe/.dockercfg.
Login Succeeded

$ echo $(cat ~/.dockercfg)
{ "https://index.docker.io/v1/": { "auth": "ZmFrZXBhc3N3b3JkMTIK", "email": "jdoe@example.com" } }

$ cat ~/.dockercfg | base64
eyAiaHR0cHM6Ly9pbmRleC5kb2NrZXIuaW8vdjEvIjogeyAiYXV0aCI6ICJabUZyWlhCaGMzTjNiM0prTVRJSyIsICJlbWFpbCI6ICJqZG9lQGV4YW1wbGUuY29tIiB9IH0K

$ cat > /tmp/image-pull-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: myregistrykey
data:
  .dockercfg: eyAiaHR0cHM6Ly9pbmRleC5kb2NrZXIuaW8vdjEvIjogeyAiYXV0aCI6ICJabUZyWlhCaGMzTjNiM0prTVRJSyIsICJlbWFpbCI6ICJqZG9lQGV4YW1wbGUuY29tIiB9IH0K
type: kubernetes.io/dockercfg
EOF

$ kubectl create -f ./image-pull-secret.yaml
secrets/myregistrykey
```

现在，你就可以创建通过imagePullSecrets引用secret的Pod：

```
apiVersion: v1
kind: Pod
metadata:
  name: foo
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: myregistrykey
```

### 4、辅助容器

Pods提供了集中运行多个容器的能力，它们可以在主机上被用来做应用的垂直整合。   
但是Pods设计的原始动机是为协助主应用程序提供帮助，典型的例子有data pullers、data pushers和proxies。

这些代表性的容器经常通过文件系统和其他容器进行交互，将相同的卷挂载到不同的容器中就可以做到这一点。   
这种模式的一个例子是使用git来进行代码管理的的web服务：

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  template:
    metadata:
      labels:
        app: nginx
    spec:
      volumes:
      - name: www-data
        emptyDir: {}
      containers:
      - name: nginx
        image: nginx
        # This container reads from the www-data volume
        volumeMounts:
        - mountPath: /srv/www
          name: www-data
          readOnly: true
      - name: git-monitor
        image: myrepo/git-monitor
        env:
        - name: GIT_REPO
          value: http://github.com/some/repo.git
        # This container writes to the www-data volume
        volumeMounts:
        - mountPath: /data
          name: www-data
```
 
更多的使用例子在：   
[blog article](http://blog.kubernetes.io/2015/06/the-distributed-system-toolkit-patterns.html)   
[presentation slides](http://www.slideshare.net/Docker/slideshare-burns)

### 5、资源分配

k8s的Scheduler只会在拥有足够的CPU和内存的节点上布置应用，但是要做到这一点，Scheduler就必须知道它们需要多少资源。

如果指定的CPU资源太少，当有很多容器被启动在同一个节点上的时候就会出现CPU匮缺。   
同样的，当没有足够的内存在请求的时候，容器将会因为不可预料的内存溢出而挂掉，尤其是需要大内存的应用。

如果没有明确指定资源需求，名义上，资源分配的数量是默认的（这个默认值是通过默认Namespace下的LimitRange应用的，可以通过kubectl describe limitrange limits来查看）。   
你可以通过以下的方式明确地指定资源的需求量：

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        resources:
          limits:
            # cpu units are cores
            cpu: 500m
            # memory units are bytes
            memory: 64Mi
          requests:
            # cpu units are cores
            cpu: 500m
            # memory units are bytes
            memory: 64Mi
```

容器请求的资源超出规定的限制之后会出现内存溢出的错误而挂掉，所以指定的值略高于预期准备指定的值会普遍提高可靠性。   
通过规定资源的需求量，保证了Pod在需要的时候可以有足够的资源来使用。   
更多资源限制和资源需求量的设置请看：   
[Resource QoS ](https://github.com/kubernetes/kubernetes/blob/master/docs/proposals/resource-qos.md)

如果你不确定多少资源需要分配，刚开始你可以不指定多少资源分配直接启动应用，之后通过
[resource usage monitoring](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/monitoring.md)
来观察和确定出合适的值。

### 6、活跃度和状态信息检查（又名健康状态检查）

许多应用运行了很长一段时间后有可能会因为很多原因最后变为不可用的，除了重启之外无法恢复还原。
k8s提供了
[liveness probes](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/pod-states.md#container-probes)
用来检查和补救以上的情况。

可以使用HTTP进行常规的应用检查，定义如下：

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            # Path to probe; should be cheap, but representative of typical behavior
            path: /index.html
            port: 80
          initialDelaySeconds: 30
          timeoutSeconds: 1
```

至于其他的时间，应用可能只是暂时无法提供服务，并且会进行自我恢复。   
这个时候，你肯定不愿去直接关掉应用，但是也不要给他发送请求，因为它不会马上回复你甚至根本不响应。   
一个典型的场景是当应用初始化的时候加载大量数据或者配置文件。

k8s提供了Readiness probes来检查和缓解这种情况，Readiness probes的配置和Liveness probes大致是相同的，只是使用readinessProbe字段。   
Pod中的容器将会通过k8s服务来报告它们现在是否准备就绪。

更多详细请看：

[example in the walkthrough](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/walkthrough/k8s201.md#health-checking)   
[standalone example](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/liveness)   
[documentation](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/pod-states.md#container-probe)

### 7、生命周期中的钩子和终止注意

节点和应用可能在任何时间点失败挂掉，但是对于干净地关闭，对很多应用来说都是有益的。   
例如当故意关掉应用的时候，完成正在处理中的请求。   
k8s提供了两种通知来实现：   
k8s将会给应用发送SIGTERM信号，可以用来正确、优雅地关闭应用。当应用没有提前终止的时候，SIGKILL将会发送一个配置好的时间数（默认为30秒，通过spec.terminationGracePeriodSeconds配置）。   
k8s提供了[pre-stop lifecycle hook](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/container-environment.md#container-hooks)的配置声明，将会在发送SIGTERM之前执行。

pre-stop hook的配置方式和probes大致相同，但是没有timing-related参数，例如：

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
        lifecycle:
          preStop:
            exec:
              # SIGTERM triggers a quick exit; gracefully terminate instead
              command: ["/usr/sbin/nginx","-s","quit"]
```

### 8、终止信息

实现一个高可用的机制是必要的，尤其是对于一些十分活跃的应用。   
快速DEBUG是十分重要的，k8s当出现致命的错误时可以快速debug，并且在kubectl或者UI中显示，除了一般的日志收集。   
一个容器挂掉之后通过terminationMessagePath配置“写下它的遗言”这是有可能的，就像打印错误、异常和堆栈信息一样。   
默认的路劲为/dev/termination-log。

以下是一个小例子：

```
apiVersion: v1
kind: Pod
metadata:
  name: pod-w-message
spec:
  containers:
  - name: messager
    image: "ubuntu:14.04"
    command: ["/bin/sh","-c"]
    args: ["sleep 60 && /bin/echo Sleep expired > /dev/termination-log"]
```

这些消息的最后部分会使用其他的规定来单独存储：

```shell
$ kubectl create -f ./pod.yaml
pods/pod-w-message
$ sleep 70
$ kubectl get pods/pod-w-message -o go-template="{{range .status.containerStatuses}}{{.lastState.terminated.message}}{{end}}"
Sleep expired
$ kubectl get pods/pod-w-message -o go-template="{{range .status.containerStatuses}}{{.lastState.terminated.exitCode}}{{end}}"
0
```

## 二、管理部署

现在你已经并通过Service与外界相连接。   
那么现在可以准备怎么办呢？   
k8s提供了一系列的工具帮助你管理应用的部署，包括扩展和更新。   
除了这些，我们将会在   
[configuration files](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/configuring-containers.md#configuration-in-kubernetes)   
和   
[labels](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/deploying-applications.md#labels)   
中讨论更多的细节。

### 1、组织资源的配置

很多应用要求创建许多资源，例如RC和SVC。   
可以将这些资源在同一个文件中分组来简化对他们的管理（在YAML文件中通过 --- 来分隔）。   
如下面的例子：

```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-svc
  labels:
    app: nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: nginx
---
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```

多个资源的创建方式和之前的创建单个资源一样：

```shell
$ kubectl create -f ./nginx-app.yaml
services/my-nginx-svc
replicationcontrollers/my-nginx
```

这些资源将会按照文件中的顺序来创建。   
所以，第一个定义SVC是最好的，因为这样一来Scheduler就可以使这个RC创建的Pods都通过SVC来连接。

kubectl create命令也接受多个-f参数：

```shell
$ kubectl create -f ./nginx-svc.yaml -f ./nginx-rc.yaml
```

也可以指定一个目录而不仅仅是一个文件：

```shell
$ kubectl create -f ./nginx/
```

kubectl将会读取所有后缀名为.yaml、.yml、.json的文件。

这是一个推荐的做法，来把一个微服务或者应用中相关的资源在一个文件中顺序定义，并且在一个目录中将这些和你的应用相关的文件分组。   
如果每层中的应用都通过DNS互相连接绑定，那么你就可以十分简单地部署所有组件。

一个URL也可以作为配置文件的源，例如直接使用签入git中的配置文件来部署：

```shell
$ kubectl create -f https://raw.githubusercontent.com/GoogleCloudPlatform/kubernetes/master/docs/user-guide/replication.yaml
replicationcontrollers/nginx
```

### 2、Kubectl中的批量操作

kubectl并不只是仅仅能够执行批量创建资源的操作。   
它也可以将资源的名字从配置文件中抽取出来，用来进行一些其他的操作，特别是要删除你创建的相同的资源的时候：

```shell
$ kubectl delete -f ./nginx/
replicationcontrollers/my-nginx
services/my-nginx-svc
```

在这里例子中，只有两个资源被删除了，同样的效果在命令行中也可以通过资源的名称很容易地操作：

```shell
$ kubectl delete replicationcontrollers/my-nginx services/my-nginx-svc
```

对于资源数量很大的情况，另外一种是可以使用Labels来过滤资源。   
Selector通过-l参数来使用：

```shell
$ kubectl delete all -lapp=nginx
replicationcontrollers/my-nginx
services/my-nginx-svc
```

因为kubectl会使用相同的语法将其接受的资源名称再次输出，所以通过$()或者xarg可以很容易地进行连续的操作：

```shell
$ kubectl get $(kubectl create -f ./nginx/ | grep my-nginx)
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR    REPLICAS
my-nginx     nginx          nginx      app=nginx   2
NAME           LABELS      SELECTOR    IP(S)          PORT(S)
my-nginx-svc   app=nginx   app=nginx   10.0.152.174   80/TCP
```

### 3、有效地使用Labels

目前为止我们使用的例子中，资源最多有一个Label。   
下面介绍在资源集合之间使用多个Labels来进行区分。

例如，不同的应用使用了不同的label，app等于不同的值，但是一个多层的应用，例如[guestbook example](https://github.com/kubernetes/kubernetes/blob/master/examples/guestbook)
需要额外地区分不同层，那么前端可以携带以下标签：

```
labels:
    app: guestbook
    tier: frontend
```

而Redis主节点和从节点使用不同的tier标签，同时甚至有可能会有一个role标签：

```
labels:
    app: guestbook
    tier: backend
    role: master

labels:
    app: guestbook
    tier: backend
    role: slave
```

Labels允许我们可以将资源划分成任意级别，然后根据这些划分尺度进行交互：

```shell
$ kubectl create -f ./guestbook-fe.yaml -f ./redis-master.yaml -f ./redis-slave.yaml
replicationcontrollers/guestbook-fe
replicationcontrollers/guestbook-redis-master
replicationcontrollers/guestbook-redis-slave
$ kubectl get pods -Lapp -Ltier -Lrole
NAME                           READY     STATUS    RESTARTS   AGE       APP         TIER       ROLE
guestbook-fe-4nlpb             1/1       Running   0          1m        guestbook   frontend   <n/a>
guestbook-fe-ght6d             1/1       Running   0          1m        guestbook   frontend   <n/a>
guestbook-fe-jpy62             1/1       Running   0          1m        guestbook   frontend   <n/a>
guestbook-redis-master-5pg3b   1/1       Running   0          1m        guestbook   backend    master
guestbook-redis-slave-2q2yf    1/1       Running   0          1m        guestbook   backend    slave
guestbook-redis-slave-qgazl    1/1       Running   0          1m        guestbook   backend    slave
my-nginx-divi2                 1/1       Running   0          29m       nginx       <n/a>      <n/a>
my-nginx-o0ef1                 1/1       Running   0          29m       nginx       <n/a>      <n/a>
$ kubectl get pods -lapp=guestbook,role=slave
NAME                          READY     STATUS    RESTARTS   AGE
guestbook-redis-slave-2q2yf   1/1       Running   0          3m
guestbook-redis-slave-qgazl   1/1       Running   0          3m
```

### 4、多种部署

另外一个会使用到多标签的场景是：分别部署一个组件的不同版本、不同配置。   
将应用的新版本和旧版一同部署是十分正常的，这样一来，在旧版本淘汰之前，新版应用可以接管旧版本的一些功能使用。   
例如，一个新版的guestbook前端可以携带以下的标签：

```
labels:
    app: guestbook
    tier: frontend
    track: canary
```

同时，稳定版本的track标签有不同的值，这样一来，两个RC控制的Pods集合就不会发生重叠：

```
labels:
    app: guestbook
    tier: frontend
    track: stable
```

通过选择它们Labels的子集，省略track这个Label，前端的Service可以跨越两组replicas：

```
selector:
 app: guestbook
 tier: frontend
```

### 5、更新Labels

有时候，在创建新的资源之前，已存在的Pods或者其他资源需要重置Label。   
这可以通过kubectl label来做到，例如：

```shell
$ kubectl label pods -lapp=nginx tier=fe
NAME                READY     STATUS    RESTARTS   AGE
my-nginx-v4-9gw19   1/1       Running   0          14m
NAME                READY     STATUS    RESTARTS   AGE
my-nginx-v4-hayza   1/1       Running   0          13m
NAME                READY     STATUS    RESTARTS   AGE
my-nginx-v4-mde6m   1/1       Running   0          17m
NAME                READY     STATUS    RESTARTS   AGE
my-nginx-v4-sh6m8   1/1       Running   0          18m
NAME                READY     STATUS    RESTARTS   AGE
my-nginx-v4-wfof4   1/1       Running   0          16m
$ kubectl get pods -lapp=nginx -Ltier
NAME                READY     STATUS    RESTARTS   AGE       TIER
my-nginx-v4-9gw19   1/1       Running   0          15m       fe
my-nginx-v4-hayza   1/1       Running   0          14m       fe
my-nginx-v4-mde6m   1/1       Running   0          18m       fe
my-nginx-v4-sh6m8   1/1       Running   0          19m       fe
my-nginx-v4-wfof4   1/1       Running   0          16m       fe
```

### 6、扩缩你的应用

当你的应用负载增加或者缩小了，通过kubectl可以十分简单地进行扩缩。   
例如，将nginx replicas的Pod数量由2增长到3：

```shell
$ kubectl scale rc my-nginx --replicas=3
scaled
$ kubectl get pods -lapp=nginx
NAME             READY     STATUS    RESTARTS   AGE
my-nginx-1jgkf   1/1       Running   0          3m
my-nginx-divi2   1/1       Running   0          1h
my-nginx-o0ef1   1/1       Running   0          1h
```

### 7、保证服务不中断的情况下更新你的应用

有时候，你需要更新你已经部署的应用，在上面的部署例子中，是通过一个新的Image或者Image tag来更新。   
kubectl不同的更新操作方式，适用于各种不同的场景。

在保证服务不中断的情况下，为了更新应用，kubectl提供了一个叫做“rolling update”的操作，它是一次更新一个Pod，而不是在同一时间将所有服务停掉。   
请查看   
[rolling update design document](https://github.com/kubernetes/kubernetes/blob/master/docs/design/simple-rolling-update.md)   
和   
[example of rolling update](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/update-demo)   
来获得更多信息。

假设你运行着1.7.9版本的nginx：

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx
spec:
  replicas: 5
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```

为了将其升级到1.9.1，你可以使用kubectl rolling-update --image：

```shell
$ kubectl rolling-update my-nginx --image=nginx:1.9.1
Creating my-nginx-ccba8fbd8cc8160970f63f9a2696fc46
```

打开另外一个窗口，你可以看到kubectl为了将新的Pods和旧的区分开，为这些Pods添加了一个deployment标签，值为配置的Hash值：

```shell
$ kubectl get pods -lapp=nginx -Ldeployment
NAME                                              READY     STATUS    RESTARTS   AGE       DEPLOYMENT
my-nginx-1jgkf                                    1/1       Running   0          1h        2d1d7a8f682934a254002b56404b813e
my-nginx-ccba8fbd8cc8160970f63f9a2696fc46-k156z   1/1       Running   0          1m        ccba8fbd8cc8160970f63f9a2696fc46
my-nginx-ccba8fbd8cc8160970f63f9a2696fc46-v95yh   1/1       Running   0          35s       ccba8fbd8cc8160970f63f9a2696fc46
my-nginx-divi2                                    1/1       Running   0          2h        2d1d7a8f682934a254002b56404b813e
my-nginx-o0ef1                                    1/1       Running   0          2h        2d1d7a8f682934a254002b56404b813e
my-nginx-q6all                                    1/1       Running   0          8m        2d1d7a8f682934a254002b56404b813e
```

kubectl rolling-update会将进展过程报告出来：

```shell
Updating my-nginx replicas: 4, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 1
At end of loop: my-nginx replicas: 4, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 1
At beginning of loop: my-nginx replicas: 3, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 2
Updating my-nginx replicas: 3, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 2
At end of loop: my-nginx replicas: 3, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 2
At beginning of loop: my-nginx replicas: 2, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 3
Updating my-nginx replicas: 2, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 3
At end of loop: my-nginx replicas: 2, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 3
At beginning of loop: my-nginx replicas: 1, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 4
Updating my-nginx replicas: 1, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 4
At end of loop: my-nginx replicas: 1, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 4
At beginning of loop: my-nginx replicas: 0, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 5
Updating my-nginx replicas: 0, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 5
At end of loop: my-nginx replicas: 0, my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 replicas: 5
Update succeeded. Deleting old controller: my-nginx
Renaming my-nginx-ccba8fbd8cc8160970f63f9a2696fc46 to my-nginx
my-nginx
```

如果你中途遇到错误，可以使用--rollback参数来终止rolling update并恢复到以前的版本：

```shell
$ kubectl kubectl rolling-update my-nginx  --image=nginx:1.9.1 --rollback
Found existing update in progress (my-nginx-ccba8fbd8cc8160970f63f9a2696fc46), resuming.
Found desired replicas.Continuing update with existing controller my-nginx.
Stopping my-nginx-02ca3e87d8685813dbe1f8c164a46f02 replicas: 1 -> 0
Update succeeded. Deleting my-nginx-ccba8fbd8cc8160970f63f9a2696fc46
my-nginx
```

不变的容器是最好的，可以通过这个例子来保持容器不变。

如果你想更新的不仅仅是镜像（如，命令参数和环境变量），你可以用一个新的name和label来创建一个新的RC：

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: my-nginx-v4
spec:
  replicas: 5
  selector:
    app: nginx
    deployment: v4
  template:
    metadata:
      labels:
        app: nginx
        deployment: v4
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.2
        args: [“nginx”,”-T”]
        ports:
        - containerPort: 80
```

然后将原本的Pod更新到新版：

```shell
$ kubectl rolling-update my-nginx -f ./nginx-rc.yaml
Creating my-nginx-v4
At beginning of loop: my-nginx replicas: 4, my-nginx-v4 replicas: 1
Updating my-nginx replicas: 4, my-nginx-v4 replicas: 1
At end of loop: my-nginx replicas: 4, my-nginx-v4 replicas: 1
At beginning of loop: my-nginx replicas: 3, my-nginx-v4 replicas: 2
Updating my-nginx replicas: 3, my-nginx-v4 replicas: 2
At end of loop: my-nginx replicas: 3, my-nginx-v4 replicas: 2
At beginning of loop: my-nginx replicas: 2, my-nginx-v4 replicas: 3
Updating my-nginx replicas: 2, my-nginx-v4 replicas: 3
At end of loop: my-nginx replicas: 2, my-nginx-v4 replicas: 3
At beginning of loop: my-nginx replicas: 1, my-nginx-v4 replicas: 4
Updating my-nginx replicas: 1, my-nginx-v4 replicas: 4
At end of loop: my-nginx replicas: 1, my-nginx-v4 replicas: 4
At beginning of loop: my-nginx replicas: 0, my-nginx-v4 replicas: 5
Updating my-nginx replicas: 0, my-nginx-v4 replicas: 5
At end of loop: my-nginx replicas: 0, my-nginx-v4 replicas: 5
Update succeeded. Deleting my-nginx
my-nginx-v4
```

你也可以通过[update demo](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/update-demo)来查看一个可视化的rolling update程序。

### 8、就地更新资源

有时候，对你创建的资源进行一些小的、无干扰的更新是必要的。   
例如你可能想为你的Object添加描述信息，[annotation](https://github.com/kubernetes/kubernetes/blob/master/docs/user-guide/annotations.md)。   
通过kubectl patch命令可以很轻松的做到：

```shell
$ kubectl patch rc my-nginx-v4 -p '{"metadata": {"annotations": {"description": "my frontend running nginx"}}}' 
my-nginx-v4
$ kubectl get rc my-nginx-v4 -o yaml
apiVersion: v1
kind: ReplicationController
metadata:
  annotations:
    description: my frontend running nginx
...
```

这个patch使用JSON格式来定义的。

对于一些更重要的更新，你可以使用get命令查看资源，编辑之后使用replace命令来更新资源并带有版本号：

```shell
$ kubectl get rc my-nginx-v4 -o yaml > /tmp/nginx.yaml
$ vi /tmp/nginx.yaml
$ kubectl replace -f /tmp/nginx.yaml
replicationcontrollers/my-nginx-v4
$ rm $TMP
```

系统通过确认当前resourceVersion和你所做编辑的版本是不一样的，来确保你不会破坏别的用户或者组件所做的更新。   
如果你想不顾一切地更新，在编辑的时候将resourceVersion移除。   
但是，如果你想这么做，不要使用你的原始配置文件作为更新源，因为追加的字段最好是在资源处于活动状态时设置。

### 9、暂时性的更新

在有些情况下，你可能需要更新一些不能一次就初始化完毕的资源，或者你可能只想要做完更改之后马上恢复原样。   
例如修复一些RC创建的不可用的Pods，可以通过replace --force删除和重新创建资源来做到这些。
假设这样子的话，你可以简单的修改你的原始配置文件：

```shell
$ kubectl replace -f ./nginx-rc.yaml --force
replicationcontrollers/my-nginx-v4
replicationcontrollers/my-nginx-v4
```