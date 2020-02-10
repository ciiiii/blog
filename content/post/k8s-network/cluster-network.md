---
title: "K8S集群网络浅析"
date: 2020-01-23T20:59:10+08:00
draft: false
description: "kubernetes network"
keywords: ["kubernetes", "kube-proxy", "NodePort", "Ingress", "LoadBalancer", "kubeDNS", "CoreDNS"]
tags: ["kubernetes"]
categories: ["concept"]
---

## 概述

K8S官方文档[集群网络](https://kubernetes.io/docs/concepts/cluster-administration/networking/)描述了实现K8S网络的四个要点：

1. 高度耦合的容器间通信
2. Pod间通信
3. Pod到Service的通信
4. 外部到Service的通信
<!--more-->
>1. Highly-coupled container-to-container communications
>2. Pod-to-Pod communications
>3. Pod-to-Service communications
>4. External-to-Service communications

## 解析
### 容器间通信
在这种情况下，一个Pod下的两个容器是共享存储、网络的，所以只需要通过localhost来进行互相访问。
### Pod间通信
Pod间通信的基础是：

1. 集群内的每一个Pod都拥有一个独立的IP
2. 任意Pod可以通过IP互连

K8S是通过CNI(Container Network Interface)插件来实现这一扁平化的网络模型的，主要有以下几种：

+ Flannel
+ Calico
+ Weave Net

具体的原理和实现暂时略过，后续会在CNI专题中详细展开。
<!-- *ps：需要注意的是，CNI插件并不只负责IP的分配和连通，* -->
### Pod到Service的通信
#### 访问方式：

+ 集群内容器通过 *\<service\>.\<namspace\>:\<port\>* 访问跨namespace的service
+ 集群内容器通过 *\<service\>:\<port\>* 访问同namespace的service

#### 主要过程：

##### DNS解析：
在容器中，根据 */etc/resolve.conf* 中的配置进行解析，nameserver为集群DNS服务的ClusterIP，search域则是域名匹配的顺序。
```
nameserver 10.254.25.10
search default.svc.cluster.local svc.cluster.local cluster.local
```
根据该search域，匹配步骤如下(target为访问域名)：

1. target.*default.svc.cluster.local*
2. target.*svc.cluster.local*
3. target.*cluster.local*

每一步拼接得到的域名，都会查询集群DNS服务，若记录存在，则直接返回对应Service的ClusterIP。

##### Service负载均衡(非Headless Service)：
上一步DNS解析中得到了Service的ClusterIP，映射到每一个Pod的PodIP的工作则是由kube-proxy负责的。

集群中的每个Node都运行了kube-proxy服务，它订阅了API server中的Service和Endpoint对象，一旦发生变化，就会修改本机的iptables规则。

当容器内访问Service的ClusterIP时，会被宿主机的iptables修改为对应Pods中任一Pod的IP，从而实现Pod到Service的通信。

### 外部到Service的通信
#### NodePort: 
##### 通过监听每个Node上的固定端口来暴露Service
当有NodePort类型的Service被创建时，kube-proxy就会创建对应的iptables规则，使访问\<NodeIP\>:\<NodePort\>的流量转发到对应的\<PodIP\>:\<ContainerPort\>上。

这种方式一般需要结合外部的LB来使用。

#### LoadBalancer: 
##### 通过Cloud Provider创建外部LB来暴露Service
LoadBalancer类型的Service绑定的外部LB把流量转发到\<NodeIP\>:\<NodePort\>上，本质上与NodePort类型的Service没有区别，只是自动创建并配置了外部LB。

#### Ingress: 
##### 通过Ingress Controller来路由暴露Service
上述两种暴露方式都只能暴露单一Service，如果一个集群中有若干Service需要对外暴露，Ingress的暴露方式会更合适。

不同于上述两种方式，只需要指定Service的类型，这里引入了Ingress资源：

+ 同时集群内还部署了一个Ingress Controller服务，并订阅了API server中的Ingress和Endpoint对象；
+ 当一个Ingress创建时，Ingress Controller会根据Ingress配置添加路由规则；
+ Ingress这种模式通过配置不同的路由把集群内多个Service通过一个Service来暴露出去。

## 总结
网络是K8S的核心，里面的每个部分都值得深入剖析，本文仅仅概述了其中个关键点，今后还会对CNI、kube-proxy、CoreDNS等组件详细分析。




