<!--
# Copyright (c) 2018-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# Fleet Command 部署：NVIDIA Triton 推理服务器

本文提供了一个用于在 Fleet Command 上安装单个 NVIDIA Triton 推理服务器集群的 helm chart。默认情况下，集群包含一个 Triton 实例，但可以通过设置 *replicaCount* 配置参数来创建任意大小的集群，具体说明如下。

本指南假设您已经部署了一个功能正常的 Fleet Command 位置。请参考 [Fleet Command 文档](https://docs.nvidia.com/fleet-command/prod_fleet-command/prod_fleet-command/overview.html)

以下步骤描述了如何设置模型仓库、使用 helm 启动 Triton，然后向运行中的 Triton 推理服务器发送推理请求。您可以选择使用 Prometheus 抓取指标，并访问 Grafana 端点来查看 Triton 报告的实时指标。

## 模型仓库

如果您已经有模型仓库，可以将其用于此 helm chart。如果您没有模型仓库，可以检出 Triton 推理服务器源代码仓库的本地副本来创建示例模型仓库：

```
$ git clone https://github.com/triton-inference-server/server.git
```

Triton 需要一个可用于推理的模型仓库。在本例中，您将把模型仓库放在 S3 存储桶中（可以是 AWS 或其他兼容 S3 API 的本地对象存储）。

```
$ aws s3 mb s3://triton-inference-server-repository
```

按照[快速入门](../../docs/getting_started/quickstart_cn.md)指南下载示例模型仓库到您的系统，并将其复制到 AWS S3 存储桶中。

```
$ aws s3 cp -r docs/examples/model_repository s3://triton-inference-server-repository/model_repository
```

### AWS 模型仓库

要从 AWS S3 加载模型，您需要将以下 AWS 凭证转换为 base64 格式，并在创建 Fleet Command 部署时将其添加到应用程序配置部分。

```
echo -n 'REGION' | base64
echo -n 'SECRECT_KEY_ID' | base64
echo -n 'SECRET_ACCESS_KEY' | base64
# 可选，用于使用会话令牌
echo -n 'AWS_SESSION_TOKEN' | base64
```

## 部署 Triton 推理服务器

通过创建部署将 Triton 推理服务器部署到您的 Fleet Command 位置。您可以在应用程序配置部分指定配置参数来覆盖默认的 [values.yaml](values.yaml)。

*注意：* 您 _必须_ 提供一个 `--model-repository` 参数，其中包含 S3 存储桶中已准备好的模型仓库的路径。否则，Triton 将无法启动。

Fleet Command 上 Triton 的示例应用程序配置：
```yaml
image:
  serverArgs:
    - --model-repository=s3://triton-inference-server-repository

secret:
  region: <region in base 64 >
  id: <access id in base 64 >
  key: <access key in base 64>
  token: <session token in base 64 (optional)>
```

更多信息请参见 [Fleet Command 文档](https://docs.nvidia.com/fleet-command/prod_fleet-command/prod_fleet-command/ug-deploying-to-the-edge.html)。

### Prometheus ServiceMonitor 支持

如果您已部署 `prometheus-operator`，可以通过在应用程序配置中设置 `serviceMonitor.enabled: true` 来启用 Triton 推理服务器的 ServiceMonitor。这还将部署一个作为 ConfigMap 的 Triton Grafana 仪表板。

否则，可以通过将外部 Prometheus 实例指向 values 中的 `metricsNodePort` 来抓取指标。

## 使用 Triton 推理服务器

现在 Triton 推理服务器已经运行，您可以向它发送 HTTP 或 GRPC 请求来执行推理。默认情况下，服务使用 NodePort 服务类型公开，在位置中的所有系统上打开相同的端口。

Triton 在端口 30343 上公开 HTTP 端点，在端口 30344 上公开 GRPC 端点，在端口 30345 上公开 Prometheus 指标端点。这些端口可以在部署时在应用程序配置中覆盖。您可以使用 curl 从 HTTP 端点获取 Triton 的元数据。例如，如果您位置中的系统 IP 为 `34.83.9.133`：

```
$ curl 34.83.9.133:30343/v2
```

按照[快速入门](../../docs/getting_started/quickstart_cn.md)指南获取示例图像分类客户端，该客户端可用于使用 Triton 提供的图像分类模型执行推理。例如：

```
$ image_client -u 34.83.9.133:30343 -m densenet_onnx -s INCEPTION -c 3 mug.jpg
Request 0, batch size 1
Image '/workspace/images/mug.jpg':
    15.349568 (504) = COFFEE MUG
    13.227468 (968) = CUP
    10.424893 (505) = COFFEEPOT
```