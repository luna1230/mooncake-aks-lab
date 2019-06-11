# 

## 热身运动

### 1.1 登录Azure公有云账号

本文将通过Azure公有云的Kubernetes服务Azure Kubernetes Service（AKS）快速地创建一个Kubernetes集群。

### 1.2 创建带有container 的虚拟机
从界面上选择创建带有container 的虚拟机。这里我们选择 Windows Server 2019 Datacenter with Containers

在左上角选择create a reasource ，搜索container ，在搜索结果中选择Windows Server 2019 Datacenter with Containers ，点击创建
<p align="left">
	<img src="https://github.com/luna1230/mooncake-aks-lab/blob/master/images/window_basic.jpg" alt="Sample"  width="490" height="670">
</p>
<p align="left">
	<img src="https://github.com/luna1230/mooncake-aks-lab/blob/master/images/window_disk.jpg" alt="Sample"  width="490" height="670">
</p>
<p align="left">
	<img src="https://github.com/luna1230/mooncake-aks-lab/blob/master/images/window_network.jpg" alt="Sample"  width="490" height="670">
</p>
<p align="left">
	<img src="https://github.com/luna1230/mooncake-aks-lab/blob/master/images/window_verify.jpg" alt="Sample"  width="490" height="670">
</p>

### 1.3 关闭安全管控
通过RDP登录到虚拟机中，关闭IE的安全管控
<p align="left">
	<img src="https://github.com/luna1230/mooncake-aks-lab/blob/master/images/window_IE.jpg" alt="Sample"  width="700" height="400">
</p>

### 1.4 本地安装Azure CLI

这时可以在本地机器安装Azure CLI，详细安装部署请参考下面的连接。

> Azure CLI安装文档： https://docs.microsoft.com/zh-cn/cli/azure/install-azure-cli-windows?view=azure-cli-latest 

完成安装后打开powershell 输入命令`az -v`可以查看Azure CLI的版本。

```
$ az -v
```

### 1.4 通过Azure CLI登录Azure

```
$ az cloud set --name AzureChinaCloud
$ az login  --use-device-code
To sign in, use a web browser to open the page https://microsoft.com/deviceloginchina and enter the code CQGSMCJGP to authenticate
```

打开浏览器输入地址[https://microsoft.com/deviceloginchina](https://microsoft.com/deviceloginchina)，填入code进行登陆

登录成功会显示

```
[
  {
    "cloudName": "AzureChinaCloud",
    "id": "7e287aa7-991f-4f81-ad20-d629a2cd6524",
        ...
    }
  }
]
```

## 2 创建Kubernetes集群

### 2.1 通过AKS创建Kubernetes集群

执行如下命令在Azure上创建一个Kubernetes集群。

```
$ az group create -n k8s-cloud-labs -l chinaeast2
$ az aks create -g k8s-cloud-labs -n k8s-cluster --disable-rbac --generate-ssh-keys
```

第一条命令是创建一个资源组，可以认为这是Azure上的存放对象的文件夹。这里我们选择使用East US数据中心。

第二条命令是真正创建Kubernetes集群的命令。`az aks`命令是操作AKS服务的子命令。我们在资源组`k8s-cloud-labs`中创建了一个名为`k8s-cluster`的集群。命令执行后，稍等片刻。5-10分钟后一个生产可用的Kubernetes将会就绪。

执行如下命令可以查看当前Azure账号下已经创建好的Kubernetes集群列表

```
$ az aks list -o table
```

### 2.2 访问Kubernetes集群

Kubernetes集群就绪后，下一步就可以通过Kubernetes的命令行`kubectl`对集群进行访问。读者可以自己到Kubernetes的GitHub主页上下载对应版本的kubectl。也可以在本地环境中直接执行如下命令自动下载并安装kubectl。

```
$ az aks install-cli
```

kubectl安装完毕后，执行如下命令获取Kubernetes集群的连接信息和访问密钥。

```
$ az aks get-credentials -g k8s-cloud-labs -n k8s-cluster
Merged "k8s-cluster" as current context in /home/luna/.kube/config
```

在cmd 命令行中设置kubectl 的执行路径

```
$ set PATH=%PATH%;C:\Users\luna\.azure-kubectl
```

一切就绪后，便可以通过`kubectl`命令操作Kuberentes集群。

```
$ kubectl get nodes
NAME                       STATUS    ROLES     AGE       VERSION
aks-nodepool1-16810059-0   Ready     agent     10m       v1.9.11
aks-nodepool1-16810059-1   Ready     agent     10m       v1.9.11
aks-nodepool1-16810059-2   Ready     agent     10m       v1.9.11
```

通过输出看到命令列出了Kubernetes集群的节点列表。`az aks create`命令默认创建了一个含有3个计算节点的集群。通过参数`--node-count`用户可以指定集群计算节点（node）的数量。目前单集群最大支持100个节点。

### 2.3 部署一个Nginx容器

通过AKS，我们快速地获取了一个生产可用的Kubernetes集群，接下来我们将部署一个简单的Nginx容器到刚创建好的Kubernetes集群。

执行如下命令创建一个Kubernetes的命名空间。

```
$ kubectl create namespace lab01
```

在新创建的命名空间下部署一个Nginx应用。命令和示例输出如下：

```
$ kubectl run frontend --image nginx:1.15 -n lab01
deployment.apps/frontend created
```

稍等片刻后可以看到Nginx的容器被成功地部署并运行。

```
$ kubectl get pod -n lab01
NAME                        READY     STATUS    RESTARTS   AGE
frontend-54fccfb9d4-kb5pl   1/1       Running   0          2m
```

### 2.4 对外发布服务

为了访问这个Nginx容器所提供的HTTP服务，我们创建一个Kubernetes Service对象。执行命令如下：

```
$ kubectl expose deployment frontend --type LoadBalancer --port 80 -n lab01
```

上文指定Service的类型为`LoadBalancer`，这样Azure将会为这个Service分配一个公网IP地址。稍等片刻，查看刚创建的Service时便可以看到`EXTERNAL-IP`一栏中显示了该Service的公网IP地址。

```
$ kubectl get svc -n lab01
NAME       TYPE           CLUSTER-IP    EXTERNAL-IP      PORT(S)        AGE
frontend   LoadBalancer   10.0.90.138   137.135.82.191   80:31650/TCP   3m
```

通过命令`curl`或浏览器访问Service的公网IP地址便可以看到Nginx服务返回的HTML代码。

```
$ curl 137.135.82.191
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

提示！因为Kubernetes在Azure公有云之上，因此可以很便捷地通过LoadBalancer类型的Service将运行在Kubernetes集群上的容器应用服务发布到互联网上。除了Service之外，AKS还提供了Application Routing这一Ingress功能，帮助用户快速将应用发布到互联网上，对外提供服务。

### 2.5 管理控制台

AKS提供的是经过测试的原生的Kubernetes集群，默认也部署了Kubernetes Dashboard。用户在本地主机上可以通过如下命令直接打开Dashboard。

```
 az aks browse -g k8s-cloud-labs -n k8s-cluster
```

## 3 Kubernetes的应用部署和管理

在前面的实验里面，我们通过命令`kubectl run`部署了一个Nginx容器。在实际的工作环境中，应用的部署往往涉及多个组件和复杂的参数配置。用户需要一个高效工具来提升应用部署和管理的效率。Helm是Kubernetes的软件包管理工具。类似Linux世界的RPM和APT，Helm可以帮助用户方便地部署和管理Kubernetes集群上的容器软件。

Helm是Microsoft团队创建的开源项目。目前，Helm也已经是CNCF的项目成员。Helm已经成为了Kubernetes社区最受欢迎的应用部署和管理工具！

### 3.1 安装Helm

可以从Helm的GitHub主页上下载Helm的二进制文件，并放置于系统环境变量`PATH`所指定的可执行文件搜索路径中。

> 提示！点击下载Helm：[https://github.com/helm/helm/releases](https://github.com/helm/helm/releases)

### 3.2 配置Helm

在使用Helm管理Kuberentes集群的应用以前，需要执行以下命令进行集群的初始化配置。

```
$ set path=C:\Users\luna\Downloads\helm-v3.0.0-alpha.1-windows-amd64\windows-amd64;%path%
$ helm init
```

### 3.3 部署WordPress应用

执行命令`helm install`安装一个WordPress应用。

```
$ helm install stable/wordpress --namespace lab02
```

执行部署后，通过命令`helm list`看查看到当前集群已经部署的容器应用。

```
$ helm list
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
mothy-bobcat    1               Sat Dec  1 16:03:38 2018        DEPLOYED        wordpress-4.0.0 4.9.8           lab02
```

### 3.4 数据持久化

WordPress应用包含了一个前端和一个MariaDB的后端。MariaDB的数据通过Persistent Volume进行了持久化。查看Persistent Volume Claim，可以看到相关的配置。

```
$ kubectl get pvc -n lab02
NAME                          STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-mothy-bobcat-mariadb-0   Bound     pvc-aaad63e8-f582-11e8-818c-bad14be25da1   8Gi        RWO            default        1m
mothy-bobcat-wordpress        Bound     pvc-aa905ca9-f582-11e8-818c-bad14be25da1   10Gi       RWO            default        1m
```

AKS管理的Kubernetes集群默认已经与Azure Disk进行了集成。当用户在Kubernetes中创建PVC时，Azure Disk PV将会自动创建并分配给相应的PVC。通过命令`az disk`可以查看到自动创建的Azure Disk磁盘。

```
$ az disk list -o table |grep -i k8s-cloud-labs|grep pvc
kubernetes-dynamic-pvc-aa905ca9-f582-11e8-818c-bad14be25da1         MC_K8S-CLOUD-LABS_K8S-CLUSTER_EASTUS                       eastus
Standard_LRS               10        Succeeded
kubernetes-dynamic-pvc-aaad63e8-f582-11e8-818c-bad14be25da1         MC_K8S-CLOUD-LABS_K8S-CLUSTER_EASTUS                       eastus
Standard_LRS               8         Succeeded
```

### 3.5 访问应用

部署后稍等片刻，可以看到成功启动的WordPress的容器。

```
$ kubectl get pod -n lab02
NAME                                      READY     STATUS    RESTARTS   AGE
mothy-bobcat-mariadb-0                    1/1       Running   0          4m
mothy-bobcat-wordpress-868c995d88-r8fdp   0/1       Running   0          4m
```

Helm部署时同时创建了相应的Service对象。

```
$ kubectl get svc -n lab02
NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)                      AGE
mothy-bobcat-mariadb     ClusterIP      10.0.136.135   <none>         3306/TCP                     5m
mothy-bobcat-wordpress   LoadBalancer   10.0.22.40     40.87.91.243   80:32485/TCP,443:30817/TCP   5m
```

通过WordPress的Service LoadBalancer的公网IP地址就可以访问到WordPress应用。如上面的例子，通过浏览器访问http://40.87.91.243/

### 3.6 应用日志和监控管理

Kubernetes的日志和监控指标的收集可以通过开源的Fluentd、Elastic Search，Kibana及Prometheus实现。Azure提供了Azure Monitor服务可以对Kubernetes集群的容器日志和监控指标进行收集和展示。通过下面的命令可以为Kubernetes集群开启Azure Monitor的支持。

```
$ az aks enable-addons --addons monitoring -g k8s-cloud-labs -n k8s-cluster
```

服务开启后，可以在Azure的Web控制台中查看容器的日志和监控信息。Azure提供了丰富和直观的查询和展示功能。

> Azure Monitor开启后，需要等待片刻，待完成初始的数据采集后方会返回查询结果和展示图表信息。

### 3.7 Kubernetes集群的伸缩

在公有云上使用Kubernetes一个优势就在于可以利用公有云庞大的计算资源。AKS提供了Kubernetes集群的弹性伸缩，通过简单的命令或者界面操作，用户就可以便捷地为Kubernetes集群添加或者删除节点。

查看当前Kubernetes集群的节点信息。通过下面的示例输出可以看到，当前集群有3个节点。节点的虚拟机规格是`Standard_DS2_v2`，操作系统为Linux。

```
$ az aks list --query "[?name=='k8s-cluster'].agentPoolProfiles[0]" -o table
Name       Count    VmSize           OsDiskSizeGb    StorageProfile    MaxPods    OsType
---------  -------  ---------------  --------------  ----------------  ---------  --------
nodepool1  3        Standard_DS2_v2  30              ManagedDisks      110        Linux
```

通过命令`az aks scale`用户可以对集群进行伸缩。下面的例子是将集群扩展至4个节点。

```
$ az aks scale -g k8s-cloud-labs -n k8s-cluster -c 4
```

命令执行后，稍等片刻。在此查看集群节点状态便可以看到集群已经成功扩展至4个节点。

```
$ az aks list --query "[?name=='k8s-cluster'].agentPoolProfiles[0]" -o table
Name       Count    VmSize           OsDiskSizeGb    StorageProfile    MaxPods    OsType
---------  -------  ---------------  --------------  ----------------  ---------  --------
nodepool1  4        Standard_DS2_v2  30              ManagedDisks      110        Linux
```

除了快速为集群扩容外，也可以对集群进行缩容。在业务空闲时间，通过缩容，用户可以节省不必要的开销。在下面的例子里，我们将集群的节点缩减至2个。

```
$ az aks scale -g k8s-cloud-labs -n k8s-cluster -c 2
```

缩容操作完成后检查集群节点，可以看到集群节点数已减少至2个。

```
$ az aks list --query "[?name=='k8s-cluster'].agentPoolProfiles[0].count"
Result
--------
2
```

## 4 Kuberentes的容器应用开发

### 4.1 下载应用代码

下载并安装git 客户端

> 点击下载Git：[https://git-scm.com/downloads](https://git-scm.com/downloads)

下载本实验所使用的示例代码。该应用是一个基于Spring Boot的RESTful应用。

```
$ mkdir git-repo && cd git-repo
$ git clone https://github.com/luna1230/japp-spring-boot-rest.git japp 
```

### 4.2 应用容器化

要把这个Java应用在Kubernetes上运行起来，首先需要先将应用进行容器化。一般而言，开发人员需要编写Dockerfile。为了方便部署，也需要编写和准备相应的Helm Chart。这个应用容器化的工作，可以手工完成，但是也可以借助工具来提高效率。Draft就是一个帮助应用容器化的工具。Draft是Helm的创作团队的又一力作。

### 4.3 下载Draft

可以在Draft的主页上下载其二进制执行文件。并将其放置于环境变量PATH所指定的二进制搜索路径中。

> 点击下载Draft：[https://github.com/Azure/draft/releases](https://github.com/Azure/draft/releases)

### 4.4 配置Draft

在Powershell 中解压Draft 安装文件

```
$ cd C:\\Users\\luna\\Downloads && tar draft-v0.16.0-windows-amd64.tar.tar
```

Draft解压完毕后，在cmd 命令行中执行

```
$ set path=C:\Users\luna\Downloads\windows-amd64;%path%
$ draft init
```

执行命令`draft init`，Draft将自动配置本地的环境，下载需要的文件和配置。

Draft下载完毕后，执行命令`draft init`，Draft将自动配置本地的环境，下载需要的文件和配置。

### 4.5 创建容器化配置

执行命令`draft create`生成应用容器化所需的Dockerfile和Helm Chart等配置。Draft可以根据代码的类型自动选择生成的模板。

```
$ cd japp
$ draft create
```

Draft将在应用的目录下生成许多配置相关的文件。

```
$ ls -a
./   .dockerignore  .draft-tasks.toml  .mvn/    Dockerfile  mvnw*     pom.xml
../  .draftignore   .gitignore         charts/  draft.toml  mvnw.cmd  src/
```

编辑draft.toml将属性`namespace`修改为`lab03`。这样后续应用将会被部署到命名空间`lab03`里。

```
namespace = "lab03"
```

### 4.6 镜像仓库

应用容器化后将会生成容器镜像。容器镜像的存放也是一个需要重点关注的问题。用户可以将镜像存放在公共的镜像仓库或私有的镜像仓库中。Azure Container Registry（ACR）是Azure提供的一个镜像仓库服务。ACR是一个提供企业级安全和全球镜像同步功能的镜像仓库。

执行以下命令，创建一个ACR仓库。本示例使用的镜像仓库名称为`k8scloudlabs`，请根据实际情况选择可用的镜像仓库名称。

```
$ az acr create -g k8s-cloud-labs -n k8scloudlabs --sku Standard
NAME          RESOURCE GROUP    LOCATION    SKU       LOGIN SERVER             CREATION DATE         ADMIN ENABLED
------------  ----------------  ----------  --------  -----------------------  --------------------  ---------------
k8scloudlabs  k8s-cloud-labs    eastus      Standard  k8scloudlabs.azurecr.io  2018-12-02T11:59:09Z


$az acr login -g k8s-cloud-labs -n k8scloudlabs
```

### 4.7 授权访问镜像仓库

ACR是一个带有安全认证机制的镜像仓库，AKS访问ACR下载镜像仓库前需要先进行授权。复制并执行下面命令，获取需要的信息，并执行授权。

```
AKS_RESOURCE_GROUP=k8s-cloud-labs
AKS_CLUSTER_NAME=k8s-cluster
ACR_RESOURCE_GROUP=k8s-cloud-labs
ACR_NAME=k8scloudlabs

CLIENT_ID=$(az aks show -g $AKS_RESOURCE_GROUP -n $AKS_CLUSTER_NAME --query "servicePrincipalProfile.clientId" -o tsv)
ACR_ID=$(az acr show -n $ACR_NAME -g $ACR_RESOURCE_GROUP --query "id" -o tsv)
az role assignment create --assignee $CLIENT_ID --role Reader --scope $ACR_ID
```

> 提示！在Windows桌面中运行`az role`命令出错的情况下，可以将如下命令的输出拷贝至CMD中执行。

```
$ echo az role assignment create --assignee $CLIENT_ID --role Reader --scope $ACR_ID
```

### 4.8 云端容器镜像构建

ACR不单止提供容器镜像的存取服务，ACR还提供基于云端的容器镜像构建服务。这意味着开发人员甚至不需要在本地开发环境安装Docker就可以进行容器镜像的构建。

执行以下命令，让Draft使用ACR进行容器镜像的构建。

```
$ draft config set container-builder acrbuild
$ draft config set registry  k8scloudlabs.azurecr.io
$ draft config set resource-group-name k8s-cloud-labs
```

### 4.9 执行应用容器镜像构建

创建所需的命名空间。

```
$ kubectl create ns lab03
```

一切配置就绪，下面将执行构建。执行命令`draft up`。Draft将把本地的代码提交给ACR，ACR将在Azure公有云上进行镜像的构建。

```
$ cd japp
$draft up
Draft Up Started: 'japp': 01CXQGZKF88NDGGKGY4C93P58A
japp: Building Docker Image: SUCCESS ⚓  (136.1632s)
japp: Releasing Application: SUCCESS ⚓  (6.9090s)
Inspect the logs with `draft logs 01CXQGZKF88NDGGKGY4C93P58A`
```

> 提示！如果构建过程出错，可以尝试执行命令`az login`及`az acr login`重新登陆Azure及ACR服务。

构建结束后，查看ACR镜像仓库的镜像列表，便可以看到相应的镜像已经成功生成。

```
$az acr repository list -g k8s-cloud-labs -n k8scloudlabs
Result
--------
japp
```

容器镜像构建完毕后，Draft还将调用Helm，将应用部署至Kubernetes集群中。通过Helm可以查看到应用部署的情况。

```
$helm list
NAME            REVISION        UPDATED                         STATUS          CHART           APP VERSION     NAMESPACE
calico-ostrich  1               Sun Dec  2 21:54:59 2018        DEPLOYED        japp-v0.1.0                     lab03
```

### 4.10 访问远程容器服务

当应用容器运行后，通过命令`draft up`可以将远程容器的端口映射到本地进行访问。

```
$draft connect
Connect to japp:4567 on localhost:59698
...内容省略...
```

通过命令`curl`访问Draft监听的端口，即可访问远程的容器应用。

```
curl http://localhost:59698/greeting?name=Kubernetes
```

### 4.11 运行本地变更

当本地代码有变化，可以重新运行一次命令`draft up`，这样Draft将根据最新的代码变更进行镜像构建和部署。
