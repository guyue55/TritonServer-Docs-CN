<!--
# Copyright (c) 2018-2023, NVIDIA CORPORATION. All rights reserved.
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

# Kubernetes部署：Triton推理服务器集群

本项目提供了一个用于安装单个Triton推理服务器集群的helm chart。默认情况下，集群包含一个推理服务器实例，但可以通过设置*replicaCount*配置参数来创建任意大小的集群，具体说明如下。

本指南假设您已经有一个功能正常的Kubernetes集群和已安装的helm（有关安装helm的说明，请参见下文）。请注意以下要求：

* helm chart部署了Prometheus和Grafana来收集和显示Triton指标。要使用此helm chart，您必须按照下面的说明在集群中安装Prometheus和Grafana，并且您的集群必须包含足够的CPU资源来支持这些服务。

* 如果您希望Triton服务器使用GPU进行推理，您的集群必须配置所需数量的GPU节点（推荐使用EC2 G4实例），并支持您所使用的推理服务器版本所需的NVIDIA驱动程序和CUDA版本。

以下步骤描述了如何设置模型仓库、使用helm启动推理服务器，然后向运行中的服务器发送推理请求。您可以访问Grafana端点查看推理服务器报告的实时指标。

## 安装Helm

### Helm v3

如果您的Kubernetes集群中尚未安装Helm，执行[官方helm安装指南](https://helm.sh/docs/intro/install/)中的以下步骤将为您提供快速设置。

如果您当前正在使用Helm v2并希望迁移到Helm v3，请参阅[官方迁移指南](https://helm.sh/docs/topics/v2_v3_migration/)。

### Helm v2

> **注意**：从现在开始，此chart将仅针对Helm v3进行测试和维护。

以下是安装Helm v2的示例说明。

```
$ curl https://raw.githubusercontent.com/helm/helm/master/scripts/get | bash
$ kubectl create serviceaccount -n kube-system tiller
serviceaccount/tiller created
$ kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
$ helm init --service-account tiller --wait
```

如果遇到任何问题，您可以参考[这里](https://v2.helm.sh/docs/install/)的官方安装指南。

## 模型仓库

如果您已经有模型仓库，可以将其用于此helm chart。如果您没有模型仓库，可以检出推理服务器源代码仓库的本地副本来创建示例模型仓库：

```
$ git clone https://github.com/triton-inference-server/server.git
```

Triton服务器需要一个可用于推理的模型仓库。在此示例中，您将把模型仓库放在AWS S3存储桶中。

```
$ aws s3 mb s3://triton-inference-server-repository
```

按照[快速入门](../../docs/getting_started/quickstart_cn.md)的说明，下载示例模型仓库到您的系统，并将其复制到AWS S3存储桶中。

```
$ aws s3 cp --recursive docs/examples/model_repository s3://triton-inference-server-repository/model_repository
```

### AWS模型仓库
要从AWS S3加载模型，您需要将以下AWS凭证转换为base64格式并添加到values.yaml中

```
echo -n 'REGION' | base64
```
```
echo -n 'SECRECT_KEY_ID' | base64
```
```
echo -n 'SECRET_ACCESS_KEY' | base64
```

## 部署Prometheus和Grafana

推理服务器指标由Prometheus收集并可通过Grafana查看。推理服务器helm chart假设Prometheus和Grafana可用，因此即使您不想使用Grafana，也必须执行此步骤。

使用[kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)安装这些组件。需要*serviceMonitorSelectorNilUsesHelmValues*标志，以便Prometheus能够在下面部署的*example*发布中找到推理服务器指标。

```
$ helm install example-metrics --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false prometheus-community/kube-prometheus-stack
```

然后端口转发到Grafana服务，以便您可以从本地浏览器访问它。

```
$ kubectl port-forward service/example-metrics-grafana 8080:80
```

现在您应该能够在浏览器中导航到localhost:8080并看到Grafana登录页面。使用用户名=admin和密码=prom-operator登录。

dashboard.json中提供了一个示例Grafana仪表板。使用Grafana中的导入功能导入并查看此仪表板。

## 部署推理服务器

使用以下命令使用默认配置部署推理服务器。

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

有几种方法可以覆盖默认配置，如[helm文档](https://helm.sh/docs/using_helm/#customizing-the-chart-before-installing)中所述。

您可以直接编辑values.yaml文件，也可以使用*--set*选项通过CLI覆盖单个参数。例如，要部署四个推理服务器的集群，使用*--set*设置replicaCount参数。

```
$ helm install example --set replicaCount=4 .
```

您还可以编写自己的"config.yaml"文件，其中包含要覆盖的值，并将其传递给helm。

```
$ cat << EOF > config.yaml
namespace: MyCustomNamespace
image:
  imageName: nvcr.io/nvidia/tritonserver:custom-tag
  modelRepositoryPath: gs://my_model_repository
EOF
$ helm install example -f config.yaml .
```

## 使用Triton推理服务器

现在推理服务器正在运行，您可以向它发送HTTP或GRPC请求来执行推理。默认情况下，推理服务通过LoadBalancer服务类型公开。使用以下命令查找推理服务器的外部IP。在本例中是34.83.9.133。

```
$ kubectl get services
NAME                             TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                        AGE
...
example-triton-inference-server  LoadBalancer   10.18.13.28    34.83.9.133   8000:30249/TCP,8001:30068/TCP,8002:32723/TCP   47m
```

推理服务器在端口8000上公开HTTP端点，在端口8001上公开GRPC端点，在端口8002上公开Prometheus指标端点。您可以使用curl从HTTP端点获取推理服务器的元数据。

```
$ curl 34.83.9.133:8000/v2
```

按照[快速入门](../../docs/getting_started/quickstart_cn.md)获取示例图像分类客户端，该客户端可用于使用推理服务器提供的图像分类模型执行推理。例如，

```
$ image_client -u 34.83.9.133:8000 -m inception_graphdef -s INCEPTION -c3 mug.jpg
Request 0, batch size 1
Image 'images/mug.jpg':
    504 (COFFEE MUG) = 0.723992
    968 (CUP) = 0.270953
    967 (ESPRESSO) = 0.00115997
```

## 清理

完成使用推理服务器后，应使用helm删除部署。

```
$ helm list
NAME            REVISION  UPDATED                   STATUS    CHART                          APP VERSION   NAMESPACE
example         1         Wed Feb 27 22:16:55 2019  DEPLOYED  triton-inference-server-1.0.0  1.0           default
example-metrics	1       	Tue Jan 21 12:24:07 2020	DEPLOYED	prometheus-operator-6.18.0   	 0.32.0     	 default

$ helm uninstall example
$ helm uninstall example-metrics
```

对于Prometheus和Grafana服务，您应该[显式删除CRD](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#uninstall-helm-chart)：

```
$ kubectl delete crd alertmanagerconfigs.monitoring.coreos.com alertmanagers.monitoring.coreos.com podmonitors.monitoring.coreos.com probes.monitoring.coreos.com prometheuses.monitoring.coreos.com prometheusrules.monitoring.coreos.com servicemonitors.monitoring.coreos.com thanosrulers.monitoring.coreos.com
```

您可能还想删除创建的用于存放模型仓库的AWS存储桶。

```
$ aws s3 rm -r gs://triton-inference-server-repository
```