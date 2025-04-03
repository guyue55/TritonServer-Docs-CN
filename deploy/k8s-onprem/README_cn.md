<!--
# Copyright (c) 2018-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
#  * Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
#  * Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
#  * Neither the name of NVIDIA CORPORATION nor the names of its
#    contributors may be used to endorse or promote products derived
#    from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS ``AS IS'' AND ANY
# EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR
# PURPOSE ARE DISCLAIMED.  IN NO EVENT SHALL THE COPYRIGHT OWNER OR
# CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL,
# EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO,
# PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR
# PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY
# OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
-->

[![License](https://img.shields.io/badge/License-BSD3-lightgrey.svg)](https://opensource.org/licenses/BSD-3-Clause)

# Kubernetes部署：NVIDIA Triton推理服务器集群

本仓库包含了在本地或AWS EC2 Kubernetes集群中安装NVIDIA Triton推理服务器的Helm chart和说明。您还可以使用本仓库为您的Triton集群启用负载均衡和自动扩展。

本指南假设您已经有一个支持GPU的功能性Kubernetes集群。有关如何安装Kubernetes并在Kubernetes集群中启用GPU访问的说明，请参阅[NVIDIA GPU Operator文档](https://docs.nvidia.com/datacenter/cloud-native/kubernetes/install-k8s.html)。您还必须安装Helm（有关说明，请参阅[安装Helm](#安装helm)）。请注意以下要求：

* 要部署Prometheus和Grafana来收集和显示Triton指标，您的集群必须包含足够的CPU资源来支持这些服务。

* 要使用GPU进行推理，您的集群必须配置所需数量的GPU节点，并支持您所使用的推理服务器版本所需的NVIDIA驱动程序和CUDA版本。

* 要启用自动扩展，您的集群的kube-apiserver必须启用[聚合层](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-aggregation-layer/)。这将允许水平Pod自动扩展器从prometheus适配器读取自定义指标。

此Helm chart可从[Triton推理服务器GitHub](https://github.com/triton-inference-server/server)获得。

有关Helm和Helm charts的更多信息，请访问[Helm文档](https://helm.sh/docs/)。

## 快速入门

首先，将此仓库克隆到本地机器。然后，执行以下命令：

安装helm

```
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

部署Prometheus和Grafana

```
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update
$ helm install example-metrics --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false prometheus-community/kube-prometheus-stack
```

使用默认设置部署Triton

```
helm install example ./deploy/k8s-onprem
```

## 安装Helm

### Helm v3

如果您的Kubernetes集群中尚未安装Helm，执行[官方Helm安装指南](https://helm.sh/docs/intro/install/)中的步骤将为您提供快速设置。

如果您当前正在使用Helm v2并希望迁移到Helm v3，请参阅[官方迁移指南](https://helm.sh/docs/topics/v2_v3_migration/)。

## 模型仓库

如果您已经有模型仓库，可以将其用于此Helm chart。如果您没有模型仓库，可以检出服务器源代码仓库的本地副本来创建示例模型仓库：

```
$ git clone https://github.com/triton-inference-server/server.git
```

Triton服务器需要一个可用于推理的模型仓库。在此示例中，我们使用现有的NFS服务器并将模型文件放在那里。有关其他支持的位置，请参阅[模型仓库文档](../../docs/user_guide/model_repository_cn.md)。

按照[快速入门](../../docs/getting_started/quickstart_cn.md)的说明，下载示例模型仓库到您的系统，并将其复制到NFS服务器上。然后，将NFS服务器的URL或IP地址以及模型仓库的服务器路径添加到`values.yaml`中。

## 部署Prometheus和Grafana

推理服务器指标由Prometheus收集并可通过Grafana查看。推理服务器Helm chart假设Prometheus和Grafana可用，因此即使您不想使用Grafana，也必须执行此步骤。

使用[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart安装这些组件。需要*serviceMonitorSelectorNilUsesHelmValues*标志，以便Prometheus能够在后面部署的*example*发布中找到推理服务器指标。

```
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update
$ helm install example-metrics --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false prometheus-community/kube-prometheus-stack
```

然后端口转发到Grafana服务，以便您可以从本地浏览器访问它。

```
$ kubectl port-forward service/example-metrics-grafana 8080:80
```

现在您应该能够在浏览器中导航到localhost:8080并看到Grafana登录页面。使用用户名=admin和密码=prom-operator登录。

dashboard.json中提供了一个示例Grafana仪表板。使用Grafana中的导入功能导入并查看此仪表板。

## 启用自动扩展

要启用自动扩展，请确保`values.yaml`中的autoscaling标签设置为`true`。这将执行两项操作：

1. 部署一个水平Pod自动扩展器，它将根据`values.yaml`中包含的信息扩展triton-inference-server的副本。

2. 安装[prometheus-adapter](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-adapter) helm chart，允许水平Pod自动扩展器基于prometheus的自定义指标进行扩展。

包含的配置将根据平均队列时间扩展Triton pod，如[此博客文章](https://developer.nvidia.com/blog/deploying-nvidia-triton-at-scale-with-mig-and-kubernetes/#:~:text=Query%20NVIDIA%20Triton%20metrics%20using%20Prometheus)中所述。要自定义此配置，您可以替换或添加`values.yaml`中的自定义规则列表。如果您更改自定义指标，请确保更改autoscaling.metrics中的值。

如果禁用自动扩展，Triton服务器pod的数量将设置为`values.yaml`中的minReplicas变量。

## 启用负载均衡

要启用负载均衡，请确保`values.yaml`中的loadBalancing标签设置为`true`。这将执行两项操作：

1. 通过[Traefik Helm Chart](https://github.com/traefik/traefik-helm-chart)部署Traefik反向代理。

2. 配置两个Traefik [IngressRoute](https://doc.traefik.io/traefik/providers/kubernetes-crd/)，一个用于http，一个用于grpc。这将允许Traefik服务公开两个端口，这些端口将被转发到Triton pod并在它们之间进行负载均衡。

要选择公开的端口号，或禁用http或grpc，请编辑`values.yaml`中的配置变量。

## 部署推理服务器

使用以下命令使用默认配置部署推理服务器、自动扩展器和负载均衡器。

在这里和以下命令中，我们使用名称`example`作为我们的chart。此名称将添加到helm安装期间创建的所有资源的开头。

```
$ cd <包含Chart.yaml的目录>
$ helm install example .
```

使用kubectl查看状态并等待推理服务器pod运行。

```
$ kubectl get pods
NAME                                               READY   STATUS    RESTARTS   AGE
example-triton-inference-server-5f74b55885-n6lt7   1/1     Running   0          2m21s
```

有几种方法可以覆盖默认配置，如[Helm文档](https://helm.sh/docs/using_helm/#customizing-the-chart-before-installing)中所述。

您可以直接编辑values.yaml文件，也可以使用*--set*选项通过CLI覆盖单个参数。例如，要部署最少有两个推理服务器的集群，使用*--set*设置autoscaler.minReplicas参数。

```
$ helm install example --set autoscaler.minReplicas=2 .
```

您还可以编写自己的"config.yaml"文件，其中包含要覆盖的值，并将其传递给Helm。如果您指定了"config.yaml"文件，设置的值将覆盖values.yaml中的值。

```
$ cat << EOF > config.yaml
namespace: MyCustomNamespace
image:
  imageName: nvcr.io/nvidia/tritonserver:custom-tag
  modelRepositoryPath: gs://my_model_repository
EOF
$ helm install example -f config.yaml .
```

## 探针配置

在`templates/deployment.yaml`中是Triton服务器容器的`livenessProbe`、`readinessProbe`和`startupProbe`的配置。默认情况下，Triton在启动HTTP服务器以响应探针之前加载所有模型。此过程可能需要几分钟，具体取决于模型大小。如果在`startupProbe.failureThreshold * startupProbe.periodSeconds`秒内未完成，则Kubernetes将其视为pod失败并重新启动它，最终导致无限循环重启pod，因此请确保为您的用例充分设置这些值。只有在启动探针首次成功后才会发送存活和就绪探针。

有关更多详细信息，请参阅[Kubernetes探针文档](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)和[启动探针的功能页面](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/950-liveness-probe-holdoff/README.md)。

## 使用Triton推理服务器

现在推理服务器正在运行，您可以向它发送HTTP或GRPC请求来执行推理。默认情况下，此chart部署[Traefik](https://traefik.io/)并使用[IngressRoute](https://doc.traefik.io/traefik/providers/kubernetes-crd/)在所有可用节点之间平衡请求。

要通过Traefik代理发送请求，请使用由Helm chart部署的traefik服务的集群IP。在本例中是10.111.128.124。

```
$ kubectl get services
NAME                              TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                    AGE
...
example-traefik                   LoadBalancer   10.111.128.124   <pending>     8001:31752/TCP,8000:31941/TCP,80:30692/TCP,443:30303/TCP   74m
example-triton-inference-server   ClusterIP      None             <none>        8000/TCP,8001/TCP,8002/TCP                                 74m
```

使用以下命令引用集群IP：
```
cluster_ip=`kubectl get svc -l app.kubernetes.io/name=traefik -o=jsonpath='{.items[0].spec.clusterIP}'`
```

Traefik反向代理在端口8000上公开HTTP端点，在端口8001上公开GRPC端点，在端口8002上公开Prometheus指标端点。您可以使用curl从HTTP端点获取推理服务器的元数据。

```
$ curl $cluster_ip:8000/v2
```

按照[快速入门](../../docs/getting