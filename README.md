# 1.安装

打开网址  https://github.com/argoproj/argo-workflows/releases  ，参考里面的命令安装，类似下面

## 1.1 安装服务端

```shell
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/download/v3.5.4/install.yaml
```

设置端口转发 UI

```shell
kubectl -n argo port-forward deployment/argo-server 2746:2746
```

打开  https://localhost:2746/



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



## 2.2 带参数的工作流

test2.yaml

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

