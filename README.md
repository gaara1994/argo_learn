# 1.安装

打开网址  https://github.com/argoproj/argo-workflows/releases  ，参考里面的命令安装，类似下面

## 1.1 安装服务端

```shell
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.4/install.yaml
```

```shell
# 可以看到部署了两个服务
yantao@ubuntu20:~$ kubectl get deployments -n argo
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
argo-server           1/1     1            1           7m1s
workflow-controller   1/1     1            1           7m1s
```

本地学习，修改认证方式，免登录。参考 https://argo-workflows.readthedocs.io/en/release-3.5/argo-server-auth-mode/

```
argo server -n argo --auth-mode=server
```

打开网页 https://localhost:2746



卸载方式 `kubectl delete namespace argo`

## 1.2 安装CLI程序

```shell
# Download the binary
curl -sLO https://github.com/argoproj/argo-workflows/releases/download/v3.5.4/argo-linux-amd64.gz

# Unzip
gunzip argo-linux-amd64.gz

# Make binary executable
chmod +x argo-linux-amd64

# Move binary to path
mv ./argo-linux-amd64 /usr/local/bin/argo

# Test installation
argo version
```



argo的使用方式

### 1.提交工作流（Workflow）

```shell
argo submit -n argo --watch https://raw.githubusercontent.com/argoproj/argo-workflows/main/examples/hello-world.yaml
```

上面使用的 `--watch` 标志将允许您观察工作流的运行情况以及它是否成功的状态。 工作流完成后，工作流上的监视将停止。`--watch`

```shell
Name:                hello-world-dsbhf
Namespace:           argo
ServiceAccount:      unset (will run with the default ServiceAccount)
Status:              Succeeded
Conditions:          
 PodRunning          False
 Completed           True
Created:             Fri Feb 23 16:43:07 +0800 (38 seconds ago)
Started:             Fri Feb 23 16:43:07 +0800 (38 seconds ago)
Finished:            Fri Feb 23 16:43:45 +0800 (now)
Duration:            38 seconds
Progress:            1/1
ResourcesDuration:   0s*(1 cpu),13s*(100Mi memory)

STEP                  TEMPLATE  PODNAME            DURATION  MESSAGE
 ✔ hello-world-dsbhf  whalesay  hello-world-dsbhf  29s 
```



### 2.列出当前工作流

```shell
argo list
```

```shell
NAME                           STATUS    AGE   DURATION   PRIORITY   MESSAGE
steps2-pqcwg                   Pending   23h   0s         0          
steps-dcnmr                    Pending   23h   0s         0          
hello-world-parameters-r2hvp   Pending   1d    0s         0  
```



### 3.获取特定工作流信息

```shell
argo get hello-world-parameters-r2hvp
```

```shell
Name:                hello-world-parameters-r2hvp
Namespace:           default
ServiceAccount:      unset (will run with the default ServiceAccount)
Status:              Pending
Created:             Thu Feb 22 16:06:24 +0800 (1 day ago)
Progress:            
Parameters:          
  message:           goodbye world

```



### 4.**打印工作流日志**

```shell
argo logs hello-world-parameters-r2hvp
```



### 5.删除工作流

```shell
argo delete hello-world-parameters-r2hvp
```

删除所有工作流

```shell
argo delete $(argo list -n argo --output name) -n argo
```



# 2.练习

我们将使用下面的镜像来练习argo的使用

```shell
yantao@ubuntu20:~/go/src/argo_learn$ sudo docker run docker/whalesay cowsay "这是传入的参数"
 _______________________ 
< 这是传入的参数 >
 ----------------------- 
    \
     \
      \     
                    ##        .            
              ## ## ##       ==            
           ## ## ## ##      ===            
       /""""""""""""""""___/ ===        
  ~~~ {~~ ~~~~ ~~~ ~~~~ ~~ ~ /  ===- ~~~   
       \______ o          __/            
        \    \        __/             
          \____\______/ 
```



## 2.1 简单的工作流

下面，我们使用 Argo 工作流模板在 Kubernetes 集群上运行相同的容器。

```shell
# 创建模版文件
vim test1.yaml
```

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow                  # new type of k8s spec
metadata:
  generateName: hello-world-    # name of the workflow spec
spec:
  entrypoint: whalesay          # invoke the whalesay template
  templates:
    - name: whalesay              # name of the template
      container:
        image: docker/whalesay
        command: [ cowsay ]
        args: [ "参数1" ]
        resources: # limit the resources
          limits:
            memory: 32Mi
            cpu: 100m

```

```
这段 YAML 文件定义了一个 Argo Workflows 工作流。下面是对其中注释的详细解释：

1. `apiVersion: argoproj.io/v1alpha1`：指定资源类型及其对应的 API 版本，这里是 Argoproj 的 Workflow 资源，版本为 v1alpha1。

2. `kind: Workflow`：声明这是一个 Kubernetes 中新的资源类型——Workflow，即 Argo Workflows 的工作流定义。

3. `metadata` 部分：
   - `generateName: hello-world-`：定义了工作流实例生成时的名称前缀。当这个 Workflow 被提交执行时，系统会根据这个前缀自动生成一个唯一的名称（如 `hello-world-xxxxx`）。

4. `spec` 部分：
   - `entrypoint: whalesay`：设置此 Workflow 的入口点是名为 "whalesay" 的模板，表示在运行此 Workflow 时将首先执行 "whalesay" 模板。

5. `templates` 列表中定义了一个模板：
   - `- name: whalesay`：指定了模板的名称为 "whalesay"。
   - `container` 字段描述了该模板内的容器配置：
     - `image: docker/whalesay`：使用 Docker Hub 上的 "docker/whalesay" 镜像作为容器的基础镜像。
     - `command: [ cowsay ]`：在容器启动后执行 "cowsay" 命令。
     - `args: [ "参数1" ]`：向 "cowsay" 命令传递一个参数 "参数1"。
     - `resources`：限制容器可使用的资源量。
       - `limits`：定义了资源上限。
         - `memory: 32Mi`：容器的最大内存限制为 32 MiB。
         - `cpu: 100m`：容器的最大 CPU 请求和限制为 100 mCPU（即 0.1 核心）。
```

提交工作流

这将会在`argo`的空间下创建。如果不指定就会在`default`下创建。

```shell
yantao@ubuntu20:~/go/src/argo_learn$ argo submit -n argo test1.yaml 
Name:                hello-world-8xf4t
Namespace:           argo
ServiceAccount:      unset (will run with the default ServiceAccount)
Status:              Pending
Created:             Fri Feb 23 17:17:48 +0800 (now)
Progress: 
```



## 2.2 参数(parameters)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-parameters-
spec:
  # invoke the whalesay template with
  # "你好 世界！" as the argument
  # to the message parameter
  entrypoint: whalesay
  arguments:
    parameters:
    - name: message
      value: 你好 世界！

  templates:
  - name: whalesay
    inputs:
      parameters:
      - name: message       # parameter declaration
    container:
      # run cowsay with that message input parameter as args
      image: docker/whalesay
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]

```

这一次，模板采用一个名为 `{{inputs.parameters.message}}"` 的输入参数，该参数作为 传递给命令。为了引用参数，参数必须用双引号括起来，以转义 YAML 中的大括号。

```shell
yantao@ubuntu20:~/go/src/argo_learn$ argo submit -n argo test2.yaml 
Name:                hello-world-parameters-2ztj4
Namespace:           argo
ServiceAccount:      unset (will run with the default ServiceAccount)
Status:              Pending
Created:             Fri Feb 23 17:19:54 +0800 (now)
Progress:            
Parameters:          
  message:           你好 世界！
```



### **覆盖参数**

argo CLI 提供了一种覆盖用于调用入口点的参数的便捷方法。下面的命令将会把 "你好 123" 绑定到 参数`message`	

```shell
yantao@ubuntu20:~/go/src/argo_learn$ argo submit -n argo test2.yaml -p message="你好 123"
Name:                hello-world-parameters-qwljj
Namespace:           argo
ServiceAccount:      unset (will run with the default ServiceAccount)
Status:              Pending
Created:             Fri Feb 23 17:47:20 +0800 (now)
Progress:            
Parameters:          
  message:           你好 123

```

### **覆盖多个参数**

如果有多个参数需要替换，

```yaml
...
templates:
- name: my-template
  inputs:
    parameters:
    - name: message1
    - name: message2
    - name: count
    - name: threshold
  container:
    image: docker/whalesay
    command: [sh, -c]
    args: ["echo '{{inputs.parameters.message1}}'; echo '{{inputs.parameters.message2}}'; ..."]
...
```

可以把参数写到单独的yaml文件中。

```yaml
# params.yaml
message1: "This is the first message"
message2: "This is the second message"
count: 50
threshold: 80.5
```

```shell
argo submit test2.yaml --parameter-file params.yaml
```



### **覆盖入口**

如果模板有3个入口

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: hello-world-parameters-
spec:
  entrypoint: whalesay
  arguments:
    parameters:
    - name: message
      value: 你好 世界！
    - name: message2
      value: 你好 世界2！
    - name: message3
      value: 你好 世界3！
  templates:
  - name: whalesay-caps1
    inputs:
      parameters:
      - name: message       # parameter declaration
    container:
      image: docker/whalesay
      command: [cowsay]
      args: ["{{inputs.parameters.message}}"]

  - name: whalesay-caps2
    inputs:
      parameters:
        - name: message2       # parameter declaration
    container:
      image: docker/whalesay
      command: [ cowsay ]
      args: ["{{inputs.parameters.message2}}"]

  - name: whalesay-caps3
    inputs:
      parameters:
        - name: message3       # parameter declaration
    container:
      image: docker/whalesay
      command: [ cowsay ]
      args: [ "{{inputs.parameters.message3}}" ]
```



```shell
argo submit test3.yaml --entrypoint whalesay-caps2
```

这样就运行whalesay-caps2了，1和3不会运行。



spec.arguments.parameters是全局的，这对于将信息传递到工作流中的多个步骤非常有用。

例如，如果要使用在每个容器的环境中设置的不同日志记录级别来运行工作流

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: global-parameters-
spec:
  entrypoint: A
  arguments:
    parameters:
    - name: log-level
      value: INFO

  templates:
  - name: A
    container:
      image: containerA
      env:
      - name: LOG_LEVEL
        value: "{{workflow.parameters.log-level}}" 
      command: [runA]
  - name: B
    container:
      image: containerB
      env:
      - name: LOG_LEVEL
        value: "{{workflow.parameters.log-level}}"
      command: [runB]

```



## 3.步骤(steps)



```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
metadata:
  generateName: steps-
spec:
  entrypoint: hello-hello-hello

  # This spec contains two templates: hello-hello-hello and whalesay
  templates:
    - name: hello-hello-hello
      # Instead of just running a container
      # This template has a sequence of steps
      steps:
        - - name: hello1            # hello1 is run before the following steps
            template: whalesay
            arguments:
              parameters:
                - name: message
                  value: "hello1"
        - - name: hello2a           # double dash => run after previous step
            template: whalesay
            arguments:
              parameters:
                - name: message
                  value: "hello2a"
          - name: hello2b           # single dash => run in parallel with previous step
            template: whalesay
            arguments:
              parameters:
                - name: message
                  value: "hello2b"
          - name: hello2c           # single dash => run in parallel with previous step
            template: whalesay
            arguments:
              parameters:
                - name: message
                  value: "hello2c"        

    # This is the same template as from the previous example
    - name: whalesay
      inputs:
        parameters:
          - name: message
      container:
        image: docker/whalesay
        command: [cowsay]
        args: ["{{inputs.parameters.message}}"]

```

这个 YAML 文件定义了一个 Argo Workflow，它包含一个名为 `hello-hello-hello` 的主模板和一个名为 `whalesay` 的子模板。具体解释如下：

1. 主模板 `hello-hello-hello` 类型为步骤（steps）模板，表示了一系列按照指定顺序执行的任务。

   - 第一步任务是 `hello1`，它引用了子模板 `whalesay`，并传入参数 `message: "hello1"`。这意味着在执行时会运行一个显示 "hello1" 消息的容器。

   - 紧接着有两个并行任务：
     - 任务 `hello2a` 同样引用了子模板 `whalesay`，但传入的是参数 `message: "hello2a"`，所以它将并行展示 "hello2a" 消息。
     - 任务 `hello2b` 同理，也是并行地引用了 `whalesay` 子模板，并传入参数 `message: "hello2b"`，从而并行展示 "hello2b" 消息。
     - 任务 `hello2bc 同理，也是并行地引用了 `whalesay` 子模板，并传入参数 `message: "hello2c"`，从而并行展示 "hello2" 消息。

2. 子模板 `whalesay` 是一个通用的工作单元，它接收一个输入参数 `message`，并基于该参数值运行一个 `docker/whalesay` 镜像的容器，执行命令 `cowsay` 并将参数作为命令行参数传递给 `cowsay` 命令。

当您使用 `argo submit` 提交此 YAML 文件时，将会启动一个工作流实例，首先执行 `hello1` 步骤，然后同时执行 `hello2a` 、 `hello2b`和`hello2c` 步骤，每个步骤都通过调用 `whalesay` 子模板来完成其操作。

```shell
argo submit -n argo --watch test4.yaml
```

```shell
STEP            TEMPLATE           PODNAME                          DURATION  MESSAGE
 ● steps-64t9v  hello-hello-hello                                                              
 ├───✔ hello1   whalesay           steps-64t9v-whalesay-3080583803  46s                        
 └─┬─◷ hello2a  whalesay           steps-64t9v-whalesay-2648452940  12s       PodInitializing  
   ├─◷ hello2b  whalesay           steps-64t9v-whalesay-2698785797  12s       PodInitializing  
   └─◷ hello2c  whalesay           steps-64t9v-whalesay-2682008178  12s 
   
STEP            TEMPLATE           PODNAME                          DURATION  MESSAGE
 ● steps-64t9v  hello-hello-hello                                                              
 ├───✔ hello1   whalesay           steps-64t9v-whalesay-3080583803  46s                        
 └─┬─◷ hello2a  whalesay           steps-64t9v-whalesay-2648452940  1m        PodInitializing  
   ├─◷ hello2b  whalesay           steps-64t9v-whalesay-2698785797  1m        PodInitializing  
   └─✔ hello2c  whalesay           steps-64t9v-whalesay-2682008178  1m    
  
STEP            TEMPLATE           PODNAME                          DURATION  MESSAGE
 ● steps-64t9v  hello-hello-hello                                                              
 ├───✔ hello1   whalesay           steps-64t9v-whalesay-3080583803  46s                        
 └─┬─✔ hello2a  whalesay           steps-64t9v-whalesay-2648452940  6m                         
   ├─◷ hello2b  whalesay           steps-64t9v-whalesay-2698785797  6m        PodInitializing  
   └─✔ hello2c  whalesay           steps-64t9v-whalesay-2682008178  1m   
```













