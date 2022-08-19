# 在EKS上的ELB获取最终用户的真实IP地址

## 一、背景

在之前的文章主要是介绍ELB+EC2模式下，获取客户端真实IP，可参考AWS官方知识库：

[https://aws.amazon.com/cn/premiumsupport/knowledge-center/elb-capture-client-ip-addresses/](https://aws.amazon.com/cn/premiumsupport/knowledge-center/elb-capture-client-ip-addresses/)

也可参考过往的blog文章：

[https://blog.bitipcman.com/get-real-client-ip-from-nlb-and-alb/](https://blog.bitipcman.com/get-real-client-ip-from-nlb-and-alb/)

在这两篇中，主要讲解是ELB+EC2场景获取真实IP地址。如果用一个表格快速概括的话，汇总如下：

| 类型 | Target类型 | 是否直接透传 | 获取真实IP的方案 |
|:--------- |:---------|:---|:----------------------|
| NLB       | Instance | 是 | 无须额外配置 |
| NLB       | IP       | 否 | 启用 Proxy V2 Protocol |
| ALB       | Instance | 否 | 启用 X-Forwarded-For Header |
| ALB       | IP       | 否 | 启用 X-Forwarded-For Header |

那么在EKS环境上，ELB的选择又包括：NLB Service模式和ALB Ingress两种模式。其中，NLB注册目标组还有IP模式和Instance模式两种。在这几种模式下，获取真实IP地址方案与ELB+EC2场景有所差别，其原因是EKS上的aws-vpc-cni和kube-proxy负责网络流量的转发，再加上AWS Load Balancer Controller负责Ingress，所以与普通ELB直接对接EC2相比有所差异。

本文分别测试如下场景。

## 二、使用NLB+目标组IP模式获取客户端真实IP

注意：本模式只对NLB和EKS在同一个VPC内的场景生效。

### 1、构建测试容器

为了测试IP地址的正常显示，我们启动一个运行apache+php的容器，并在其中放置一个默认页面`index.php`显示客户端IP地址。其内容如下。

```
Client IP Address is: <?php printf($_SERVER["REMOTE_ADDR"]); ?>
```

将这个php容器构建好，上传到ECR容器镜像，供EKS使用。

### 2、构建YAML文件

然后为EKS构建一个yaml配置文件如下。然后测试NLB入口地址。

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: 133129065110.dkr.ecr.ap-southeast-1.amazonaws.com/phpdemo:2
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: "1"
            memory: 2G
---
apiVersion: v1
kind: Service
metadata:
  name: "service-nginx"
  annotations:
        service.beta.kubernetes.io/aws-load-balancer-name: myphpdemo
        service.beta.kubernetes.io/aws-load-balancer-type: external
        service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
        service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "2"
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
        service.beta.kubernetes.io/aws-load-balancer-attributes: load_balancing.cross_zone.enabled=true
        service.beta.kubernetes.io/aws-load-balancer-target-group-attributes: preserve_client_ip.enabled=true

spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

创建完毕后，可进入NLB的目标组，可看到Pod注册为IP模式。在NLB的属性设置页面，保留客户端IP的选项显示为`Enable`。

### 3、访问NLB入口

现在用命令行curl请求EKS生成的NLB，即可看到客户端返回的是真实IP地址。

```
admin2:~/environment $ curl myphpdemo-7ed770009c9698f4.elb.ap-southeast-1.amazonaws.com

Client IP Address is: 13.212.147.226
admin2:~/environment $ ll
```

## 三、使用NLB+目标组为Instance模式+Proxy V2协议获取客户端真实IP

### 1、构建容器并启用Proxy V2协议

容器中的输出客户端IP地址的PHP代码同上同上一步。

在本测试中，通过Apache上启用Proxy V2协议的支持。Apache2可以通过mod_remoteip模块，一键打开对Proxy协议的支持。使用Amazon Linux 2安装的Apache，则已经内置了mod_remoteip模块。如果是使用CentOS系统自带的yum安装，也应该是内置支持的。如果您是手工安装的Apache，则需要查找对应模块，是否已经编译正确并放置so文件到正确的路径下。

编辑如下配置文件。

```
vi /etc/httpd/conf/httpd.conf
```

在找到 `ServerAdmin root@localhost` 这一行。在这一行的下边，加入如下一行配置：

```
RemoteIPProxyProtocol On
```

需要注意的是，这个配置应加载于默认网站或者VirtualHost的配置段内，不能在其他位置任意填写，否则Apache会无法启动。

修改完毕后，重新build重启，让本容器内启动的apache默认打开对Proxy V2的支持。

### 2、构建YAML文件

构建一个yaml配置文件如下。

```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nlb-instance
  labels:
    app: nlb-instance
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nlb-instance
  template:
    metadata:
      labels:
        app: nlb-instance
    spec:
      containers:
      - name: nlb-instance
        image: 133129065110.dkr.ecr.ap-southeast-1.amazonaws.com/phpdemo:3
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: "1"
            memory: 2G
---
apiVersion: v1
kind: Service
metadata:
  name: "nlb-instance"
  annotations:
        service.beta.kubernetes.io/aws-load-balancer-name: nlb-instance
        service.beta.kubernetes.io/aws-load-balancer-type: external
        service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: instance
        service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "2"
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "2"
        service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "10"
        service.beta.kubernetes.io/aws-load-balancer-attributes: load_balancing.cross_zone.enabled=true
        service.beta.kubernetes.io/aws-load-balancer-target-group-attributes: proxy_protocol_v2.enabled=true

spec:
  selector:
    app: nlb-instance
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

使用这个yaml创建服务。然后测试NLB入口地址。

### 4、访问NLB查看IP地址

```
admin2:~/environment $ curl nlb-instance-d3704cf68361ea94.elb.ap-southeast-1.amazonaws.com

REMOTE_ADDR Address is: 13.214.132.220

admin2:~/environment $ 
```

访问NLB后可看到获取到正确的客户端地址。

## 四、使用ALB Ingress获取客户端真实IP

### 1、构建容器

此步骤同上一步。

### 2、部署AWS Load Blancer Controller和ALB Ingress

正常安装AWS Load Balancer Controller。请参考相关文档。

### 3、构建YAML文件

构建一个yaml配置文件如下。然后测试ALB入口地址。

```
---
apiVersion: v1
kind: Namespace
metadata:
  name: alb-ip
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: alb-ip
  name: alb-ip
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: alb-ip
  replicas: 3
  template:
    metadata:
      labels:
        app.kubernetes.io/name: alb-ip
    spec:
      containers:
      - image: 133129065110.dkr.ecr.ap-southeast-1.amazonaws.com/phpdemo:2
        imagePullPolicy: Always
        name: alb-ip
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  namespace: alb-ip
  name: alb-ip
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    app.kubernetes.io/name: alb-ip
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: alb-ip
  name: alb-ip
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: alb-ip
              port:
                number: 80
```

### 4、访问ALB Ingress查看IP地址

现在用命令行curl请求EKS生成的ALB Ingress，即可看到客户端返回的ALB的内网IP。这是正常的。

```
admin2:~/environment $ curl k8s-albip-albip-b5197d6014-1443298103.ap-southeast-1.elb.amazonaws.com

Client IP Address is: 172.31.2.80
admin2:~/environment $ 
```

### 5、修改代码改为获取X-Forward HEADER得到真实IP

为了在ALB后边的容器中能获取真实的客户端IP，需要修改容器中的php代码，将显示服务器端地址的代码改为获取名为X-Forward的HTTP HEADER。修改后的`index.php`代码如下。

```
Client IP Address is: <?php printf($_SERVER["HTTP_X_FORWARDED_FOR"]); ?>
```

再次使用curl访问ALB，即可看到代码获得IP地址显示为真实IP地址。

```
admin2:~/environment $ curl k8s-albip-albip-b5197d6014-1443298103.ap-southeast-1.elb.amazonaws.com

Client IP Address is: 13.212.147.226
admin2:~/environment $ 
```

## 五、小结

### 1、测试结论汇总

通过本文测试可以看到，EKS上获取真实客户端IP的逻辑与ELB+EC2时候有所不同，汇总如下：

| 类型 | Target类型 | 是否直接透传 | EKS上Pod获取真实IP的方案 |
|:--------- |:---------|:---|:----------------------|
| NLB       | IP | 是 | 启用NLB目标组保留原始IP |
| NLB       | Instance       | 否 | 需要应用程序支持，在Apache/Nginx上启用 Proxy V2 Protocol后可获取客户端原始IP |
| ALB       | IP       | 否 | 启用 X-Forwarded-For Header |

### 2、推荐和建议

结论：考虑如下搭配组合：

* **1、使用NLB Target Group IP模式**：在这种打开保留客户端IP选项后，即可直接在EKS应用中获取客户端IP地址，步骤简单方便，推荐使用。
* **2、使用ALB Ingress模式**：此场景与普通ALB+EC2的方式相同，都是通过X_FORWARDED Header来获取真实IP地址。
* **3、使用NLB Target Group 为Instance模式**：需要应用侧额外配置Proxy V2协议。步骤相对较多，复杂。

### 3、参考文档：

[https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-target-groups.html#client-ip-preservation](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-target-groups.html#client-ip-preservation)

[https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/service/annotations/#load-balancer-attributes](https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.4/guide/service/annotations/#load-balancer-attributes)



