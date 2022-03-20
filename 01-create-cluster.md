# 实验一、创建EKS集群

EKS 1.21版本 @2022-03 Global Region测试通过

## 一、AWS CLI准备

### 1、客户端下载

本步骤对所有操作系统下都需要安装。请到[这里](https://aws.amazon.com/cli/)下载对应的操作系统的安装包。

### 2、配置AKSK和区域

安装CLI完毕后，配置进入AWS控制台，创建IAM用户，生成AKSK密钥。安装好AWSCLI，并填写正确的AKSK。同时，在命令`aws configure`的最后一步配置region的时候，设置region为本次实验的`ap-southeast-1`。

## 二、安装EKS客户端（三个OS类型根据实验者选择其一）

### 1、Windows下安装eksctl和kubectl工具

eksctl的安装可通过choco包管理工具进行。先使用管理员权限打开powershell，执行：

```
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
```

即可安装好choco。然后在cmd下用管理员权限安装eksctl和jq工具（本步骤需要管理员权限）：

```
choco install -y eksctl jq curl wget vim 7zip
```

很多日常软件都可以后续执行`choco install`安装。

Windows上的kubectl可从如下地址下载：

[https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/windows/amd64/kubectl.exe](https://amazon-eks.s3.us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/windows/amd64/kubectl.exe)

下载后请将此文件复制到 `C:\windows\system32` 目录下，由此便可在任意路径下调用。

### 2、Linux下安装eksctl和kubectl工具

在Linux下安装eks工具，包括eksctl和kubectl两个。执行如下命令：

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /bin
curl -o kubectl https://amazon-eks.s3-us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/linux/amd64/kubectl
chmod 755 kubectl
sudo mv kubectl /bin
eksctl version
```

安装完毕后即可看到eksctl版本，同时kubectl也下载完毕。

### 3、MacOS下安装eksctl和kubectl工具

先安装homebrew包管理工具。

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
curl -o kubectl  https://amazon-eks.s3-us-west-2.amazonaws.com/1.21.2/2021-07-05/bin/darwin/amd64/kubectl
chmod 755 kubectl
sudo mv kubectl /bin
eksctl version
```

## 三、创建EKS集群（两种场景二选一）

创建集群时候，eksctl默认会自动生成一个新的VPC、子网并使用192.168的网段，然后在其中创建nodegroup。如果希望使用现有VPC，请使用本章节第二种方法。

### 1、创建新VPC和子网并创建EKS集群

执行如下命令。注意如果是多人在同一个账号内实验，需要更改EKS集群的名字避免冲突。如果多人在不同账号内做实验，无需修改名称，默认的名称即可。

```
eksctl create cluster --name=eksworkshop --version=1.21 --nodes=3 --nodegroup-name=nodegroup1 --nodes-min=3 --nodes-max=6 --node-volume-size=100 --node-volume-type=gp3 --node-type m5.2xlarge --managed --alb-ingress-access --full-ecr-access --region=ap-southeast-1
```

### 2、使用现有VPC的子网创建EKS集群

如果希望使用现有VPC的Private子网，请确保本子网已经设置了正确的路由表，且VPC内包含NAT Gateway可以提供外网访问能力。

运行如下命令：

```
eksctl create cluster --name=eksworkshop --version=1.21 --nodegroup-name=nodegroup1 --nodes=3 --nodes-min=3 --nodes-max=6 --node-volume-size=100 --node-volume-type=gp3 --node-type=m5.2xlarge --managed --alb-ingress-access --full-ecr-access --vpc-private-subnets=subnet-0af2e9fc3c3ab08b4,subnet-0bb5aa110443670a1,subnet-008bcabf73bea7e58 --node-private-networking --region=ap-southeast-1
```

请替换以上命令中`--vpc-private-subnets=`后的子网ID。

### 3、查看创建结果

此过程需要10-15分钟才可以创建完毕。执行如下命令查询节点。

```
kubectl get node
```

返回节点如下表示正常。

```
NAME                                               STATUS   ROLES    AGE     VERSION
ip-192-168-76-61.ap-southeast-1.compute.internal   Ready    <none>   5m10s   v1.20.4-eks-6b7464
ip-192-168-8-47.ap-southeast-1.compute.internal    Ready    <none>   5m10s   v1.20.4-eks-6b7464
```

## 四、创建集群并配置Dashboard图形界面

### 1、部署K8S原生控制面板

Github上AWS官方Workshop的实验脚本中，采用的是直接调用Github托管的yaml文件，其域名是`raw.githubusercontent.com`。在国内网络条件下访问这个网址可能会失败。

因此本实验另外提供了另外的网址可在国内的yaml文件。请执行如下命令开始启动。

```
kubectl apply -f https://myworkshop.bitipcman.com/eks101/kubernetes-dashboard.yaml
```

部署需要等待3-5分钟。访问Dashboard的身份验证是通过token完成，执行以下命令获取token。注意需要手工替换EKS集群名称和region名称为实际操作环境。如果集群名称、Region信息不匹配，生成的token会报告401错误无法登录。

### 2、登录到Dashboard

```
aws eks get-token --cluster-name eksworkshop --region ap-southeast-1 | jq -r .status.token
```

以上命令会输出类似如下的token，稍后复制下来，登录Dashboard会使用。

```
k8s-aws-v1.aHR0cHM6Ly9zdHMuYXAtc291dGhlYXN0LTEuYW1hem9uYXdzLmNvbS8_QWN0aW9uPUdldENhbGxlcklkZW50aXR5JlZlcnNpb249MjAxMS0wNi0xNSZYLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFSSEc2NFVGR1NTT01JWDZXJTJGMjAyMjAzMTglMkZhcC1zb3V0aGVhc3QtMSUyRnN0cyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjIwMzE4VDE2NDYzMlomWC1BbXotRXhwaXJlcz02MCZYLUFtei1TaWduZWRIZWFkZXJzPWhvc3QlM0J4LWs4cy1hd3MtaWQmWC1BbXotU2lnbmF0dXJlPTFjZDkzMGZjMjNhNTI2MmIyYWNhNDlmMzM0ZTZlMTRhNzFhMDE1NzU0MjY4YjYyOTgzMzA5ZmJjYTAxZjY5NTQ
```

使用如下命令启动Proxy将Dashboard的访问映射出来。

```
kubectl proxy
```

使用Chrome等不受安全策略限制的浏览器，在实验者的本机上访问如下地址（部分Firefox受到安全策略限制访问有兼容问题）。
```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

登录页面打开后，选择第一项使用token登录，然后输入上一步获取的token，即可访问dashboard.

至此Dashboard配置完成。

### 3、删除Dashboard服务（可选）

测试完成后，如果需要删除Dashboard，执行如下命令。

```
kubectl delete -f https://myworkshop.bitipcman.com/eks101/kubernetes-dashboard.yaml
```

本命令为可选，建议保留Dashboard，在后续实验中也可以继续通过Dashboard做监控。

## 五、部署Nginx测试应用并使用NodePort+NLB模式对外暴露服务

### 1、创建服务

这个测试应用将在当前集群的两个node上创建nginx应用pod，并使用default namespace运行Service，然后通过NodePort模式和NLB对外发布在80端口。

本实验所使用的ngix-nlb.yaml配置文件与本文末尾的参考资料中Github上AWS官方Workshop内的配置有所不同，因为K8S的API版本从beta v1演进到v1，因此需要修改API Version，且加入selector配置。本实验提供的文件已经完成上述修正。

执行如下命令。

```
kubectl apply -f https://myworkshop.bitipcman.com/eks101/nginx-nlb.yaml 
```

查看创建出来的pod，执行如下命令。

```
kubectl get pods
```

返回结果如下Running表示运行正常。

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-85ff79dd56-8l8wn   1/1     Running   0          2m
```

确认部署执行如下命令。

```
kubectl get deployment nginx-deployment
```

返回结果如下，状态是Available表示工作正常。

```
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           9m12s
```

### 2、测试从浏览器访问

本实验使用的是NLB，创建NLB过程需要3-5分钟。此时可以通过AWS EC2控制台，进入Load Balance负载均衡界面，可以看到NLB处于Provisioning创建中的状态。等待其变成Active状态。接下来进入NLB的listener界面，可以看到NLB将来自80端口的流量转发到了k8s-default-servicen这个target group。点击进入Target Group，可以看到当前两个node的状态是initial，等待其健康检查完成，变成healthy状态，即可访问。

查看运行中的Service，执行如下命令。

```
kubectl get service service-nginx -o wide 
```

返回结果如下。其中的ELB域名地址就是对外访问入口。

```
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                                             PORT(S)        AGE   SELECTOR
service-nginx   LoadBalancer   10.100.224.17   acba538c728d64a8db295c3f14b0ee01-1911011890f36ddc.elb.cn-northwest-1.amazonaws.com.cn   80:31920/TCP   11m   app=nginx
```

用浏览器访问ELB地址，即可验证应用启动结果。

### 3、测试从命令行访问（可选）

也可以在命令行上通过curl命令访问。

##### Linux和MacOS操作系统如下命令是通过命令行访问：

获取NLB地址并通过curl访问：

```
NLB=$(kubectl get service service-nginx -o json | jq -r '.status.loadBalancer.ingress[].hostname')
echo $NLB
curl -m3 -v $NLB
```

##### Windows操作系统如下命令是通过命令行访问：

获取NLB地址：

```
kubectl get service service-nginx -o json | jq -r .status.loadBalancer.ingress[].hostname
```

通过CURL验证访问：

```
curl -m3 -v 上文获取到的NLB入口地址
```

由此即可访问到测试应用，看到 Welcome to nginx! 即表示访问成功。 

### 4、删除服务

执行如下命令：

```
kubectl delete -f https://myworkshop.bitipcman.com/eks101/nginx-nlb.yaml
```

至此服务删除完成。

## 六、参考文档

AWS GCR Workshop：

[https://github.com/aws-samples/eks-workshop-greater-china/tree/master/china/2020_EKS_Launch_Workshop](https://github.com/aws-samples/eks-workshop-greater-china/tree/master/china/2020_EKS_Launch_Workshop)

K8S的Dashboard安装：

[https://github.com/kubernetes/dashboard](https://github.com/kubernetes/dashboard)
