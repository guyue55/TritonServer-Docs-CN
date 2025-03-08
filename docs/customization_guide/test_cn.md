<!--
# Copyright 2018-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 测试 Triton

目前 Triton 仓库尚未启用 CI 测试。我们将在未来的更新中启用 CI 测试。

然而，在 qa/ 目录中有一组可以手动运行的测试，提供了广泛的测试覆盖。在运行这些测试之前，您必须首先生成一些包含测试所需模型的模型仓库。

## 生成 QA 模型仓库

QA 模型仓库包含一些用于验证 Triton 正确性的简单模型。要生成 QA 模型仓库：

```
$ cd qa/common
$ ./gen_qa_model_repository
$ ./gen_qa_custom_ops
```

这将在 /tmp/\<version\>/qa_* 中创建多个模型仓库（例如 /tmp/24.09/qa_model_repository）。TensorRT 模型将为 CUDA 认为是设备 0（零）的 GPU 系统创建。如果您的系统有多个 GPU，请参阅脚本中的文档了解如何针对特定 GPU。

## 构建 SDK 镜像

使用以下命令构建包含客户端库、模型分析器、性能分析器和示例的 *tritonserver_sdk* 镜像。您必须首先将 *client* 仓库的 `<client branch>` 分支检出到 clientrepo/ 子目录，并将 *perf_analyzer* 仓库的 `<perf analyzer branch>` 分支检出到 perfanalyzerrepo/ 子目录。通常，您希望将 `<client branch>` 和 `<perf analyzer branch>` 设置为与当前服务器分支相同。

```
$ cd <server repo root>
$ git clone --single-branch --depth=1 -b <client branch> https://github.com/triton-inference-server/client.git clientrepo
$ git clone --single-branch --depth=1 -b <perf analyzer branch> https://github.com/triton-inference-server/perf_analyzer.git perfanalyzerrepo
$ docker build -t tritonserver_sdk -f Dockerfile.sdk .
```

## 构建 QA 镜像

接下来，您需要构建 Triton Docker 镜像的 QA 版本。此镜像将包含 Triton、QA 测试以及运行 QA 测试所需的所有依赖项。首先执行 [Docker 镜像构建](build.md#building-with-docker)以生成 *tritonserver_cibase* 和 *tritonserver* 镜像。

然后，构建实际的 QA 镜像。

```
$ docker build -t tritonserver_qa -f Dockerfile.QA .
```

## 运行 QA 测试

现在运行 QA 镜像，并将 QA 模型仓库挂载到容器中，以便测试能够访问它们。

```
$ docker run --gpus=all -it --rm -v/tmp:/data/inferenceserver tritonserver_qa
```

在容器内，QA 测试位于 /opt/tritonserver/qa。要运行测试，请切换到测试目录并运行 test.sh 脚本。

```