## 一、快速开始

### 1、启动一个简单的容器。

一旦在container中打包好应用并将其commit为image之后，你就可以将其部署在k8s集群上。

一个简单的nginx服务器例子：

先决条件：你需要拥有的是一个部署完毕并可以正常运行的k8s集群。

在Master节点上使用kubectl命令来启动一个运行着nginx服务器的容器：

```shell
$ kubectl run my-nginx --image=nginx --replicas=2 --port=80
CONTROLLER   CONTAINER(S)   IMAGE(S)   SELECTOR       REPLICAS
my-nginx     my-nginx       nginx      run=my-nginx   2
```

以上命令会让节点上的Docker从nginx这个image上启动一个容器监听80端口，此为一个pod。   
而replicas=2则表示会起两个一模一样的pod。

使用以下命令来查看创建的pod：

```shell
$ kubectl get po
NAME             READY     STATUS    RESTARTS   AGE
my-nginx-l8n3i   1/1       Running   0          29m
my-nginx-q7jo3   1/1       Running   0          29m
```

k8s会确保你的应用是一直运行的，当容器运行失败时，k8s会自动重启容器，当整个节点失败时，会在另外一个健康的节点启动这个容器。

### 2、通过端口将应用连接到Internet上。

先决条件：拥有公网IP，或者在云服务器上，如：阿里云，亚马逊云等。

以下命令将上一步骤中的nginx容器连接到公网中：

```shell
$ kubectl expose rc my-nginx --port=80 --type=LoadBalancer
NAME       LABELS         SELECTOR       IP(S)     PORT(S)
my-nginx   run=my-nginx   run=my-nginx             80/TCP
```

rc即Replication Controller，上一步骤中的命令其实会自动创建一个名为my-nginx的rc来确保pod的数量维持在2个。   
可以使用以下命令来查看rc：

```shell
$ kubectl get rc
CONTROLLER   CONTAINER(S)  IMAGE(S)   SELECTOR      REPLICAS 
my-nginx     my-nginx      nginx      run=my-nginx     2   
```

expose命令将会创建一个service，将本地（某个节点上）的一个随机端口关联到容器中的80端口。   
可以使用以下命令来查service：

```shell
$ kubectl get svc my-nginx
NAME         LABELS          SELECTOR     IP(S)              PORT(S)         
my-nginx     run=my-nginx    run=nginx    10.254.110.117     80/TCP     
```

type指明这个svc将会起到一个负载均衡的作用，会将流量导入两个pod中。

svc会分配一个虚拟IP用来访问容器，如上步骤中分配的IP为10.254.110.117，则可以在任意节点上通过curl 10.254.110.117得到nginx的欢迎界面。   
在分配虚拟IP的过程中，你可能需要等待一些时间。

在任一节点上使用netstat -tunpl命令可以看到，kube-proxy监听的端口多了一个，端口号是随机的，可以在浏览器中输入该节点的公网IP:端口访问放nginx的欢迎界面。

### 3、删除容器。

删除rc，即删除该rc控制的所有容器。   
删除svc，即删除分配的虚拟IP。

```shell
$ kubectl delete rc my-nginx
replicationcontrollers/my-nginx
$ kubectl delete svc my-nginx
services/my-nginx
```

注意，如果使用delete pod ${podName}来删除是没有效果的，因为rc会马上启动另外一个pod来维持总数量为2。

## 二、通过配置文件来创建资源。

除了某些强制性的命令，如：kubectl run或者expose等，会隐式创建rc或者svc，k8s还允许通过配置文件的方式来创建这些操作对象。

通常，使用配置文件的方式会比直接使用命令行更可取，因为这些文件可以进行版本控制，而且文件的变化和内容也可以进行审核，当使用及其复杂的配置来提供一个稳健、可靠和易维护的系统时，这些点就显得非常重要。

在声明定义配置文件的时候，所有的配置文件都存储在YAML或者JSON格式的文件中并且遵循k8s的资源配置方式。

kubectl可以创建、更新、删除和获得API操作对象，当前apiVersion、kind和name会组成一个API Path以供kubectl来调用。

### 1、通过一个配置文件来启动容器。

k8s的pod中运行容器，一个包含简单的Hello World容器的pod可以通过YAML文件这样来定义：

```
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
spec: # 当前pod内容的声明
  restartPolicy: Never
  containers:
  - name: hello
    image: "ubuntu:14.04"
    command: ["/bin/echo","hello”,”world"]
```

创建的pod名为metadata.name的值：hello-world，该名称必须是唯一的。   
spec的内容为该pod中，各个容器的声明：

```
restartPolicy：Never表示启动后运行一次就终止这个pod。
containers[0].name为容器1的名字。
containers[0].image为该启动该容器的镜像。
containers[0].command相当于Dockerfile中定义的Entrypoint，可以通过下面的方式来声明cmd的参数：

command: ["/bin/echo"]
args: ["hello","world"]
```

使用以下命令来通过这个YAML文件创建pod：

```shell
$ kubectl create -f ./hello-world.yaml
pods/hello-world
```

### 2、检验配置文件的正确性。

当你不确定声明的配置文件是否书写正确时，可以使用以下命令要验证：

```shell
$ kubectl create -f ./hello-world.yaml --validate
```

使用--validate只是会告诉你它发现的问题，仍然会按照配置文件的声明来创建资源，除非有严重的错误使创建过程无法继续，如必要的字段缺失或者字段值不合法，不在规定列表内的字段会被忽略。

点此查看完成的字段列表：

[Pod API object](https://htmlpreview.github.io/?https://github.com/kubernetes/kubernetes/HEAD/docs/api-reference/definitions.html#_v1_pod)

### 3、环境变量和变量扩展。

因为并不是所有的image都包含有shell，所以k8s并不会自动执行shell命令，如果你想在容器的shell中执行一些命令，例如扩展环境变量，你可以这么定义配置文件：

```
apiVersion: v1
kind: Pod
metadata:
  name: hello-world
spec:  
  restartPolicy: Never
  containers:
  - name: hello
    image: "ubuntu:14.04"
    env:
    - name: MESSAGE
      value: "hello world"
    command: ["/bin/sh","-c"]
    args: ["/bin/echo \"${MESSAGE}\""]
```

但是，如果只是扩展一些环境变量就使用shell是不必要的，所以上面的声明并不会让k8s在启动pod的时候执行，如果你真的想要达到这种效果请使用以下声明代替：

```
command: ["/bin/echo"]
args: ["$(MESSAGE)"]
```

更多的设置环境变量的语法：

[$(ENVVAR) syntax](https://github.com/kubernetes/kubernetes/blob/master/docs/design/expansion.md)

### 4、查看pod状态。

你可以通过get命令来查看被创建的pod。   
如果执行完创建pod的命令之后，你的速度足够快，那么使用get命令你将会看到以下的状态：

```shell
$ kubectl get pods
NAME          READY     STATUS    RESTARTS   AGE
hello-world   0/1       Pending   0          0s
```

一个pod刚被创建的时候是不会被调度的，因为没有任何节点被选择用来运行这个pod。   
调度的过程发生在创建完成之后，但是这个过程一般很快，所以你通常看不到pod是出于unscheduler状态的除非创建的过程遇到了问题。

pod被调度之后，分配到指定的节点上运行，这时候，如果该节点没有所需要的image，那么将会自动从默认的Docker Hub上pull指定的image，一切就绪之后，你就可以看到pod是处于running状态了：

```shell
$ kubectl get pods
NAME          READY     STATUS    RESTARTS   AGE
hello-world   1/1       Running   0          5s
```

其中READY字段表示该pod中 正在运行的容器数量/总容器数量。

由于我们创建的pod是只运行一次就终止，所以在它运行起来之后，它就会马上终止了，再次查看pod状态结果如下：

```shell
$ kubectl get pods
NAME          READY     STATUS       RESTARTS   AGE
hello-world   0/1       ExitCode:0   0          15s
```

### 5、查看pod输出。

你可能会有想了解在pod中执行命令的输出是什么，和Docker logs命令一样，kubectl logs将会显示这些输出：

```shell
$ kubectl logs hello-world
hello world
```

### 6、删除pod。

当你查看完输出之后，就可以将这个pod删除了：

```shell
$ kubectl delete pod hello-world
pods/hello-world
```

和create一样，delete成功之后会打印出被删除的资源类型/资源名。

也可以使用资源类型/资源名的格式删除：

```shell
$ kubectl delete pods/hello-world
pods/hello-world
```

被终止的pod不会马上自动删除，所以你可以通过观察pod的最终状态是什么，来确认是否需要清理删除你的那些挂掉的pod。   
另外，为了释放节点上的磁盘空间，删除pod时容器和它们的日志信息会被一起自动删除。 