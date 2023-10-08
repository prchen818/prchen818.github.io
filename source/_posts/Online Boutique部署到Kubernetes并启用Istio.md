---
title: Online Boutique部署到k8s并启用istio及监控组件
date: 2023-07-09 16:58:00
categories: 教程
tags: 
- kubernetes
- 微服务
---

# 云计算实验报告

## 1、搭建 K8S 环境

### 1.1 实验条件

实验所用云服务器环境为华为云 ECS 4vCPUs 4GiB 内存，操作系统为 Ubuntu20.04，本次实验后续将使用该云服务器搭建单机 kubernetes 环境并进行应用部署。

### 1.2 安装容器运行时

k8s 集群支持的容器运行时包括 Docker 和 containerd 等，安装最新版 Docker 时也会自动安装 containerd，因此本实验中选择直接安装 Docker 作为 k8s 的容器运行时环境，步骤如下：

1. 更新软件包索引，并安装必要的依赖软件

```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
```

1. 导入源仓库的 GPG key，将 Docker APT 源添加到 apt 仓库索引中

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
```

1. 安装 Docker

```bash
sudo apt install docker-ce
```

1. 验证 Docker 版本

![](/images/B6B7boS5hovH5Wxv12kcWPEpnVg.png)

### 1.3 安装 Kubernetes（使用 KubeSphere）

由于本人此前已经使用 Kubernetes 和 KubeSphere 进行过实际项目的部署，因此这里选择直接安装 KubeSphere。

使用 KubeKey 可以方便的将 kubernetes 和 kubeSphere 一同安装，并自行选择安装 kubernetes 的可选组件。本实验中安装步骤主要参考 KubeSphere 官方文档 "**在 Linux 上以 All-in-One 模式安装 KubeSphere"**（[https://www.kubesphere.io/zh/docs/v3.3/quick-start/all-in-one-on-linux/](https://www.kubesphere.io/zh/docs/v3.3/quick-start/all-in-one-on-linux/)），具体步骤如下：

1. 安装 kubernetes 推荐的依赖项

```bash
sudo apt install socat conntrack ebtables ipset
```

1. 下载 KubeKey

```bash
export KKZONE=cn
curl -sfL https://get-kk.kubesphere.io | VERSION=v3.0.7 sh -
```

1. 安装 Kubernetes 和 KubeSphere

```bash
./kk create cluster --with-kubernetes v1.22.12 --with-kubesphere v3.3.2
```

1. 验证 KubeSphere 安装成功

```bash
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l 'app in (ks-install, ks-installer)' -o jsonpath='{.items[0].metadata.name}') -f
```

得到输出如下

![](/images/OdC0bt7JwodXVFxUY12cw5DKn3d.png)

KubeSphere 作为企业级的 K8s 运维管理平台，在安装时已经默认在 kubesphere-monitoring-system 命名空间下启用了对集群的监控，通过 KubeSphere 提供的管理页面可以方便地进行集群状态监控和应用资源监控，无需再另外安装 Dashboard。使用 admin 用户进入 KubeSphere 的集群管理页面查看集群状态如下图

![](/images/QBd4bNst5oIVhxxYPwjcBWjwnLc.png)

_后注__：由于墙和其他一些乱七八糟的 bug，在尝试重装了系统之后用同样的步骤却无法再启动 KubeShpere，导致本人心态爆炸，遂还是回到纯粹的 Kubernetes 环境进行后续实验。_

安装 Dashboard 步骤如下：

```sql
kubectl apply -f "https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml"
```

添加 Dashboard 管理员用户：

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

```bash
kubectl apply -f dash-admin-user.yml
# 获取登录Token
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

由于 Dashboard 在新版本中只允许通过 Localhost 访问，还需要通过 ssh 打洞的方式将远端的 8001 端口映射到本地：

```bash
ssh -L localhost:8001:localhost:8001 -NT root@122.112.192.212
```

再本地访问 http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/，成功打开 Dashboard 如下：

![](/images/YFcCbRf9vouIcQxXX1Lc92kCn7d.png)

## 2、Online Boutique 应用部署

### 2.1 Online Boutique 架构

从项目的 Github 首页介绍可知，Online Boutique（[https://github.com/GoogleCloudPlatform/microservices-demo](https://github.com/GoogleCloudPlatform/microservices-demo)）是一个云原生的微服务演示应用程序。该应用由 11 个微服务组成，用户可以在其中浏览商品， 将它们添加到购物车并购买。

![](/images/WfdFb4wcdoODd1xYZu5cLkj4nNb.png)

项目部署中会用到的几个主要目录中存放的内容为：

- src：11 个微服务的源代码
- kubernetes-manifests：各个微服务的配置文件
- istio-manifests：istio 的配置文件
- release：部署该项目使用到的 kubernetes-manifests.yaml 和 istio-manifests.yaml

该项目目前最新版本为 v0.8.0，相较于 v0.7.0 的主要改变在于默认启用了 gRPC 存活探针（伏笔了）

### 2.2 将 Online Boutique 部署到 Kubernetes

1. 将项目 clone 到服务器中

```bash
git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
```

1. 在 online-boutique 这个 namespace 下创建应用

```bash
cd microservice-demo/release
kubectl apply -f kukubernetes-manifests.yaml
```

部署出错：

```bash
error: error validating "kubernetes-manifests.yaml": error validating data: [ValidationError(Deployment.spec.template.spec.containers[0].livenessProbe): unknown field "grpc" in io.k8s.api.core.v1.Probe, ValidationError(Deployment.spec.template.spec.containers[0].readinessProbe): unknown field "grpc" in io.k8s.api.core.v1.Probe]; if you choose to ignore these errors, turn validation off with --validate=false
```

阅读报错信息以及 Google 查询后发现，是由于所安装的 Kubernetes 版本过低，k8s 在 v1.24Beta 版中才默认开启 grpc 探针功能，但 KubeSphere 默认安装的是 v1.22.10

_后注__：实际上对于 online boutique 这个项目，直到 v0.7.0 版本，使用 v1.22 的 kubernetes 环境已经足以用于部署，而直到最终写报告复盘时才意识到可以降低项目版本而不用执着于升级 Kubernetes 版本。就是在这里为了升级版本使得 KubeSphere 环境反复出现各种神奇的问题，最终导致本人心态爆炸全部推倒重来。_

输出如下：

```bash
root@cpr-huawei:~/microservices-demo# kubectl apply -f ./release/kubernetes-manifests.yaml
deployment.apps/emailservice created
service/emailservice created
deployment.apps/checkoutservice created
service/checkoutservice created
deployment.apps/recommendationservice created
service/recommendationservice created
deployment.apps/frontend created
service/frontend created
service/frontend-external created
deployment.apps/paymentservice created
service/paymentservice created
deployment.apps/productcatalogservice created
service/productcatalogservice created
deployment.apps/cartservice created
service/cartservice created
deployment.apps/loadgenerator created
deployment.apps/currencyservice created
service/currencyservice created
deployment.apps/shippingservice created
service/shippingservice created
deployment.apps/redis-cart created
service/redis-cart created
deployment.apps/adservice created
service/adservice created
```

但实际上通过 kubectl get pods -A 发现，几乎所有的 pod 都处于 ImagePullBackOff 的状态中，还是墙的问题

我们需要修改 kubernetes-manifests.yaml 中的镜像地址，将 gcr.io 全部替换为 gcr.lank8s.cn

重新运行 ``kubectl apply -f kukubernetes-manifests.yaml``，发现所有 pod 成功变成 running 状态。

访问<node-ip>:80 即可成功进入 online boutique 的首页，如下：

![](/images/Vt1Bb38vHoyFz4xKy9Vck5tWnVb.png)

## 3、启用 Istio 组件

### 3.1 Istio 组件原理

Istio 的官网上是这样介绍自己的：

> Istio 是一个开源服务网格，它透明地分层到现有的分布式应用程序上。 Istio 强大的特性提供了一种统一和更有效的方式来保护、连接和监视服务。 Istio 是实现负载平衡、服务到服务身份验证和监视的路径——只需要很少或不需要更改服务代码。

Istio 通过在部署的每个应用程序中添加代理“sidecar”，为用户提供流量管理，可观察性和安全性能。使用 Istio 后集群中的服务网络如下图所示。

![](/images/LfJYbvmVqoTlLvxV2j7crjirnRd.png)

Istio 的架构从逻辑上分成数据平面（Data Plane）和控制平面（Control Plane）

- 数据平面：由一组和业务服务成对出现的 Sidecar 代理（Envoy）构成，它的主要功能是接管服务的进出流量，传递并控制服务和 Mixer 组件的所有网络通信
- 控制平面：主要包括了 Pilot、Mixer、Citadel 和 Galley 共 4 个组件，主要功能是通过配置和管理 Sidecar 代理来进行流量控制，并配置 Mixer 去执行策略和收集遥测数据

### 3.2 安装 Istio

从 Github 下载支持当前 kubernetes 版本（本实验中为 v1.22）的 Istio，~~[https://github.com/istio/istio/releases/tag/1.15.7~~~](https://github.com/istio/istio/releases/tag/1.15.7~~~)~（划掉的原因见后后注）~~，解压后将文件夹上传至服务器，运行以下命令将 istio 添加到环境变量并赋予执行权限。

```bash
chmod -R 777 istio-1.15.7/
export PATH=$PWD/bin:$PATH
istioctl version
# no running Istio pods in "istio-system"
# 1.15.7

istioctl install
```

这里直接安装 Istio 会出现如下报错：

```bash
This will install the Istio 1.15.7 minimal profile with ["Istio core" "Istiod"] components into the cluster. Proceed? (y/N) y
✔ Istio core installed                                                                                                                        
✘ Istiod encountered an error: failed to wait for resource: resources not ready after 5m0s: timed out waiting for the condition               
               
- Pruning removed resources                                                                 
Error: failed to install manifests: errors occurred during operation
```

实际上似乎并不是因为超时，而是因为内存不足，即便是 minimal 安装也是如此，因此不得不换到上古版本 v1.8.6 进行尝试，然而上古版本的 istio 与当前的 kubernetes 环境并不兼容。根据 Stackoverflow 论坛的讨论，Istio 可能需要 16GiB 以上的内存才能够正常安装

_后后注__：做到这里的时候本人又又又一次心态爆炸了，200 元代金券原价购买的服务器配置，即使在仔细计算后买到了 4c4g 的不错配置，对于 Istio 所需的内存需求来说仍是杯水车薪。为了继续进行实验，不得不又一次推倒重来，使用云服务器的路走不通了，本地安装虚拟机所需的步骤又过于繁琐，所幸 Docker Desktop 自带有 Kubernetes 环境，因此后续实验直接在 Windows 环境下继续，Dashboard 直接通过 Lens 查看。_

由于环境更换到 Windows Docker Desktop，重新下载 windows 平台的 istio（[https://github.com/istio/istio/releases/download/1.18.0/istio-1.18.0-win.zip](https://github.com/istio/istio/releases/download/1.18.0/istio-1.18.0-win.zip)），并在 terminal 中使用进行安装

```bash
cd istio-1.18.0/bin/
./istioctl.exe install
```

![](/images/CYxub0SrvoLTZHxQvRQctcGcnag.png)

此外，还应将 istio/bin 路径添加到 Windows 的系统环境变量当中。

### 3.3 在 online boutique 项目中启用 Istio

将此前创建的 online boutique 应用相关资源全部删除，创建 online-boutique 命名空间，并为该 namespace 加上 istio-injection=enabled 的标签，然后再重新部署该项目

```bash
kubectl delete all -all -n default
kubectl create namespace online-boutique
kubectl label ns online-boutique istio-injection=enabled
kubectl apply -f .\release\istio-manifests.yaml -n online-boutique
```

可以看到每个 pod 变成了 2 个 container，说明 sidecar 注入成功

![](/images/CSmSbtHeIoRNSOx3kuQcHdE0nrd.png)

通过浏览器访问 online boutique 首页，访问成功

![](/images/DneqbtLwAoYsmuxLnygctCxOnSf.png)

## 4、为 Istio 配置 Kiali、Prometheus、Grafana

### 4.1 安装可选组件

在此前下载的 Istio 中已经附带了 Kiali 等组件的配置文件，我们只需要找到相应的 yaml 文件并运行即可

```bash
kubectl apply -f kiali.yaml
 kubectl apply -f promethues.yaml
 kubectl apply -f grafana.yaml
 kubectl apply -f jeager.yaml
```

![](/images/Qnalb6xpkopASOxnhOSc3lXHnye.png)

### 4.2 配置 Kiali 外部访问

Kiali 是具有服务网格配置和验证功能的 Istio 可观察性的控制台。通过监视流量来推断拓扑和错误报告，它可以帮助您了解服务网格的结构和运行状态。 Kiali 提供了详细的的指标并与 Grafana 进行基础集成，可以用于高级查询。通过与 Jaeger 来提供分布式链路追踪功能。

运行以下命令动态修改 kiali 配置

```bash
kubectl -n istio-system edit svc kiali
```

将 ClusterIP 改为 NodePort：

![](/images/PE42b7bDTo8YalxL2ZucgdJWn6d.png)

修改成功后，通过浏览器访问 Kiali，可以通过 Kiali 实现 Istio 服务网格的可视化，为 Online Boutique 项目提供服务拓扑图、全链路跟踪、指标遥测、配置校验、健康检查等功能。

![](/images/IZyFbR4Qzo4hpdxqk7ycAYtEnTg.png)

### 4.3 配置 Promethues 外部访问

Promethues 是一个开源的监控系统、时间序列数据库。我们可以利用 Prometheus 与 Istio 集成来收集指标，通过这些指标判断 Istio 和网格内的应用的运行状况。我们可以使用 Grafana 和 Kiali 来可视化这些指标。

与上一节相同，修改 Prometheus 服务的 type 为 NodePort，然后通过浏览器访问

![](/images/CzS5bAJGZoEuckxUldBcfw8En5f.png)

### 4.4 配置 Grafana 外部访问

Grafana 是一个开源的监控解决方案，可以用来为 Istio 配置仪表板。我们可以使用 Grafana 来监控 Istio 及部署在服务网格内的应用程序。

同之前一样，将 Grafana 的服务类型改为 NodePort，然后用浏览器访问，得到如下界面：

![](/images/GN59bdipjoyh7exUfr5ctxWjnJg.png)

## 5、总结

### 5.1 个人主要工作

本次实验由我个人独立完成以上报告中的所有内容。

在本次实验中，我通过 KubeKey、Kubeadm、DockerDesktop 等多种方式搭建了多个 Kubernetes 环境，将 Online Boutique 项目使用 Kubernetes 部署并成功通过公网 ip 访问到项目首页并且可以正常使用。

实验中，针对 Online Boutique 在熔断、限流、监控、认证、授权、安全、负载等方面的不足，我启用 Istio 支持，将项目升级到服务网格架构。

在配置好 Istio 后，我还为 Istio 集成了 Kiali、Prometheus、Grafana、Jaeger 等多个运维监控组件，实现了服务网格以及集群状态的可视化监控。

### 5.2 实验收获

通过这次实验，我加深了对与 Kubernetes 这个容器编排工具的理解，深入体会到了云计算技术带来的优势和便利，理解了一个微服务项目部署到 k8s 环境的全流程，学习了 Istio 服务网格的原理和使用方法，也对于云原生环境下常搭配 Istio 使用的 Kiali、Grafana 等组件有了更多的了解。

Kubernetes 作为目前最广泛使用的云原生容器编排工具，可以自动化应用程序的部署、扩展、更新和回滚，搭配 KubeSphere 或是 Dashboard 这样的可视化管理面板，可以大大减轻运维人员的工作压力，提高运维效率。同时其提供的服务发现与负载均衡功能，使得程序开发人员可以不再需要在运维上花费精力，提高程序的可用性和性能。此外，Kubernetes 提供的跨平台的能力，使得应用程序可以在不同的平台上运行，具体到这次实验来说，依靠 Kubernetes 的能力，在云服务器、虚拟机、Windows 上都可以成功运行 Online Boutique 项目，这使得应用程序的部署和迁移更加容易。

而 K8S 搭配 Istio 这个服务网格应用使用，为云原生应用提供了更高级的流量管理功能以及可靠的服务发现和负载均衡机制，能够进行更灵活的流量控制和服务治理，这使得金丝雀发布、智能路由等功能的实现轻而易举。同时 Istio 搭配 Kiali、Grafana 等附加组件，为我们提供了极为强大的可视化观测和监控功能，帮助运维人员更好的理解和调试微服务应用程序，实时监控和排查故障。

## 6、云计算技术的行业应用体会

自 Google 于 2006 年首次提出云计算的概念以来，云计算技术在工业界和学术界都得到了广泛的关注。和随着云计算、云平台的快速普及和不断发展，云计算相关技术已经给越来越多的行业带去了信息化的应用与变革。从与云计算紧紧相关的互联网行业、通信行业，到广电行业、金融行业，云计算技术已经逐渐成为各行各业进入信息化时代的关键基础设施。

云计算技术对于通信行业实现创新发展而言意义显著[1]。在云计算与大数据等相关技术的影响下，移动客户端的功能要求得以不断降低，依靠云计算技术将对于移动终端计算能力的要求转移至云平台，使得移动设备的性能要求降低，极大促进了移动互联网通信的普及。此外，通过应用云计算技术，通信行业能够提高存储和利用数据信息的能力，对移动互联网产生的海量复杂数据进行高效的存储、分析和处理，为通信行业整体信息化服务水平带来的巨大提升。

云计算依靠强大的计算能力，可实现数据储存和快速检索，有助于提升广播电视台的服务水平，为广电行业提供云平台服务[2]。云上计算资源通过专有网络或互联网进行快速提供，可以发挥其快速、弹性、持续及自动化等特点，有助于保障广播电视台的各项信息安全，并适应现代广电行业的各种运行需求，保障媒体节目得到良好的制作与播出，提升广电节目质量。依托云平台提供的服务，用户也可以在任意位置、使用各种终端获取网络媒体资源，提高便捷性和快速性。

随着数字化产业转型进程加速，云计算逐渐成为经济社会运行的数字化业务平台，大量国内金融科技、私募基金，传统银行、证券保险的创新业务已经普遍认可“业务上云”的趋势[3]。得益于云计算技术的发展，金融行业能够轻易的获取大量的算力资源，持续积累海量的数据，与人工智能等技术相结合，将云计算深入应用到数据处理、模型训练、部署运营和安全检测等产业链的全流程中。

更为重要的是，云计算对于环境保护行业以及可持续发展具有重要意义。云计算通过虚拟化的方式将计算资源进行整合，能够给用户提供弹性、动态的信息服务资源[4]，这样的方式有助于提高资源利用效率，促进可持续发展。云计算将大量应用服务集中于少数服务器，高效的资源利用和集中管理使得数据中心的能源消耗相对更低，并且减少了企业对于本地硬件设备的需求，有助于减少电子废物的产生[5]。此外，云计算的弹性和可伸缩性使得企业可以根据需求动态调整资源使用了，减少服务器和设备的闲置消耗，还可以依靠云平台和协作工具，使得员工可以远程办公写作，减少通勤消耗并提高协同效率。

通过实验与相关文献的阅读，我对于云计算、云平台技术在不同行业的应用有了更深的了解，对于云计算高可靠性、快速部署、弹性扩容、按需服务的优势和特点有了更清晰的认知，同时也意识到了云计算技术的发展对于环境保护、资源节约和可持续发展的重要意义。相信在未来随着云计算技术的不断发展，云计算将凭借其自身的优势与特点，成为现代信息化社会如同车辆与道路一样重要的基础设施，推动互联网与数字社会的可持续发展。也希望在以后的求学与研究生涯中能够继续研究与从事云计算相关工作，为云计算技术发展和应用做出自己的贡献。
