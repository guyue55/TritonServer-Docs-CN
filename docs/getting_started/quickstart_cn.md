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

# 快速入门

**刚接触 Triton Inference Server 并想快速部署您的模型？**
请使用[这些教程](https://github.com/triton-inference-server/tutorials#quick-deploy)
开始您的 Triton 之旅！

Triton Inference Server 提供[可构建的源代码](../customization_guide/build.md)，但安装和运行 Triton 最简单的方法是
使用来自 [NVIDIA GPU Cloud (NGC)](https://ngc.nvidia.com) 的预构建 Docker 镜像。

启动和维护 Triton Inference Server 主要围绕着构建模型仓库。本教程将涵盖：

* 创建模型仓库
* 启动 Triton
* 发送推理请求

## 创建模型仓库

[模型仓库](../user_guide/model_repository.md)是您存放想要 Triton 提供服务的模型的目录。
在[docs/examples/model_repository](../examples/model_repository)中包含了一个示例模型仓库。
在使用该仓库之前，您必须通过提供的脚本从其公共模型库中获取任何缺失的模型定义文件。

```
$ cd docs/examples
$ ./fetch_models.sh
```

## 启动 Triton

Triton 通过使用 GPU 进行优化以提供最佳的推理性能，但它也可以在仅 CPU 的系统上工作。
在这两种情况下，您都可以使用相同的 Triton Docker 镜像。

### 在带 GPU 的系统上运行

使用以下命令运行 Triton，并使用您刚刚创建的示例模型仓库。必须安装 [NVIDIA Container
Toolkit](https://github.com/NVIDIA/nvidia-docker) 才能让 Docker 识别 GPU。
--gpus=1 标志表示应该为 Triton 提供 1 个系统 GPU 用于推理。

```
$ docker run --gpus=1 --rm -p8000:8000 -p8001:8001 -p8002:8002 -v/full/path/to/docs/examples/model_repository:/models nvcr.io/nvidia/tritonserver:<xx.yy>-py3 tritonserver --model-repository=/models
```

其中 \<xx.yy\> 是您想要使用的 Triton 版本（并在上面已拉取）。启动 Triton 后，您将在控制台上看到
显示服务器启动和加载模型的输出。当您看到类似以下的输出时，Triton 已准备好接受推理请求。

```
+----------------------+---------+--------+
| Model                | Version | Status |
+----------------------+---------+--------+
| <model_name>         | <v>     | READY  |
| ..                   | .       | ..     |
| ..                   | .       | ..     |
+----------------------+---------+--------+
...
...
...
I1002 21:58:57.891440 62 grpc_server.cc:3914] Started GRPCInferenceService at 0.0.0.0:8001
I1002 21:58:57.893177 62 http_server.cc:2717] Started HTTPService at 0.0.0.0:8000
I1002 21:58:57.935518 62 http_server.cc:2736] Started Metrics Service at 0.0.0.0:8002
```

所有模型都应显示"READY"状态，表明它们已正确加载。如果模型加载失败，状态将报告失败和失败原因。
如果您的模型未在表中显示，请检查模型仓库的路径和您的 CUDA 驱动程序。

### 在仅 CPU 的系统上运行

在没有 GPU 的系统上，运行 Triton 时不应使用 Docker 的 --gpus 标志，但其他方面与上述相同。

```
$ docker run --rm -p8000:8000 -p8001:8001 -p8002:8002 -v/full/path/to/docs/examples/model_repository:/models nvcr.io/nvidia/tritonserver:<xx.yy>-py3 tritonserver --model-repository=/models
```

由于未使用 --gpus 标志，GPU 不可用，因此 Triton 将无法加载任何需要 GPU 的模型配置。

### 验证 Triton 是否正确运行

使用 Triton 的 *ready* 端点来验证服务器和模型是否已准备好进行推理。从主机系统使用 curl
访问指示服务器状态的 HTTP 端点。

```
$ curl -v localhost:8000/v2/health/ready
...
< HTTP/1.1 200 OK
< Content-Length: 0
< Content-Type: text/plain
```

如果 Triton 已准备就绪，HTTP 请求返回状态 200；如果未准备就绪，则返回非 200 状态。

## 发送推理请求

使用 docker pull 从 NGC 获取客户端库和示例镜像。

```
$ docker pull nvcr.io/nvidia/tritonserver:<xx.yy>-py3-sdk
```

其中 \<xx.yy\> 是您想要拉取的版本。运行客户端镜像。

```
$ docker run -it --rm --net=host nvcr.io/nvidia/tritonserver:<xx.yy>-py3-sdk
```

在 nvcr.io/nvidia/tritonserver:<xx.yy>-py3-sdk 镜像中，运行示例 image-client 应用程序，
使用示例 densenet_onnx 模型执行图像分类。

要发送 densenet_onnx 模型的请求，请使用 /workspace/images 目录中的图像。在这种情况下，
我们请求前 3 个分类结果。

```
$ /workspace/install/bin/image_client -m densenet_onnx -c 3 -s INCEPTION /workspace/images/mug.jpg
Request 0, batch size 1
Image '/workspace/images/mug.jpg':
    15.346230 (504) = COFFEE MUG
    13.224326 (968) = CUP
    10.422965 (505) = COFFEEPOT
```