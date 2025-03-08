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

# 构建 Triton

本节介绍如何从源代码构建 Triton 服务器。有关构建 Triton 客户端库和示例的信息，请参阅[客户端库和示例](https://github.com/triton-inference-server/client)。有关构建 Triton SDK 容器的信息，请参阅[构建 SDK 镜像](test.md#build-sdk-image)。有关测试 Triton 构建的信息，请参阅[测试 Triton](test.md)。

您可以创建一个自定义的 Triton Docker 镜像，其中包含已发布后端的子集，而无需从源代码构建。例如，您可能只想要一个仅包含 TensorRT 和 Python 后端的 Triton 镜像。对于这种类型的自定义，您不需要从源代码构建 Triton，而是可以使用 [*compose* 工具](compose.md)。

Triton 源代码分布在多个 GitHub 仓库中，这些仓库可以一起构建和安装，以创建完整的 Triton 安装。Triton 服务器使用 CMake 和（可选）Docker 构建。为了简化构建过程，Triton 提供了一个 [build.py](https://github.com/triton-inference-server/server/blob/main/build.py) 脚本。build.py 脚本将生成构建 Triton 所需的 CMake 和 Docker 构建步骤，并可以选择调用这些步骤或将调用留给您，如下所述。

build.py 脚本目前支持为以下平台构建 Triton。如果您尝试在此处未列出的平台上构建 Triton，请参阅[在不支持的平台上构建](#building-on-unsupported-platforms)。

* [Ubuntu 22.04, x86-64](#building-for-ubuntu-2204)

* [Jetpack 4.x, NVIDIA Jetson (Xavier, Nano, TX2)](#building-for-jetpack-4x)

* [Windows 10, x86-64](#building-for-windows-10)

如果您正在开发或调试 Triton，请参阅[开发和增量构建](#development-and-incremental-builds)以了解如何执行增量构建的信息。

## 为 Ubuntu 22.04 构建

对于 Ubuntu-22.04，build.py 同时支持 Docker 构建和非 Docker 构建。

* [使用 Docker 构建](#building-with-docker)和来自 [NVIDIA GPU Cloud (NGC)](https://ngc.nvidia.com) 的 TensorFlow 和 PyTorch Docker 镜像。

* [不使用 Docker 构建](#building-without-docker)。

### 使用 Docker 构建

构建 Triton 最简单的方法是使用 Docker。构建的结果将是一个名为 *tritonserver* 的 Docker 镜像，其中将包含 /opt/tritonserver/bin 中的 tritonserver 可执行文件和 /opt/tritonserver/lib 中所需的共享库。为 Triton 构建的后端和仓库代理将分别位于 /opt/tritonserver/backends 和 /opt/tritonserver/repoagents 中。

构建的第一步是克隆您想要构建的发布分支（或 *main* 分支以从开发分支构建）的 [triton-inference-server/server](https://github.com/triton-inference-server/server) 仓库。然后按照下面的说明运行 build.py。build.py 脚本在使用 Docker 构建时执行以下步骤。

* 在服务器仓库的 *build* 子目录中，生成构建 Triton 所需的 docker_build 脚本、cmake_build 脚本和 Dockerfile。如果您使用 --dryrun 标志，build.py 将在此处停止，以便您可以检查这些文件。