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

* helm chart部署了Prometheus和Grafana来收集和显示Triton指标。您的集群必须包含足够的CPU资源来支持这些服务。至少需要2个CPU节点，机器类型为n1-standard-2或更高。

* 如果您希望Triton服务器使用GPU进行推理，您的集群必须配置所需数量的GPU节点，并支持您所使用的推理服务器版本所需的NVIDIA驱动程序和CUDA版本。

此helm chart可从[Triton推理服务器GitHub](https://github.com/triton-inference-server/server)或[NVIDIA GPU Cloud (NGC)](https://ngc.nvidia.com)获得。

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

Triton服务器需要一个可用于推理的模型仓库。在此示例中，您将把模型仓库放在Google Cloud Storage存储桶中。

```
$ gsutil mb gs://triton-inference-server-repository
```

按照[快速入门](../../docs/getting_started/quickstart_cn.md)的说明，下载示例模型仓库到您的系统，并将其复制到GCS存储桶中。

```
$ gsutil cp -r docs/examples/model_repository gs://triton-inference-server-repository/model_repository
```

### GCS权限

确保存储桶权限设置正确，以便推理服务器可以访问模型仓库。如果存储桶是公开的，则无需额外更改，您可以直接进入"部署Prometheus和Grafana