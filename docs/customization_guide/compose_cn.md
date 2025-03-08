<!--
# Copyright (c) 2020-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 自定义 Triton 容器

[NVIDIA GPU Cloud (NGC)](https://ngc.nvidia.com) 提供了两个 Docker 镜像，使得可以轻松构建自定义版本的 Triton。通过自定义 Triton，您可以通过移除不需要的功能来显著减小 Triton 镜像的大小。

目前自定义的范围有限，如下所述，但未来的版本将增加可用的自定义选项。您也可以从源代码[构建 Triton](build.md#building-triton)以获得更精确的自定义。

## 使用 compose.py 脚本

`compose.py` 脚本可以在 [server 仓库](https://github.com/triton-inference-server/server)中找到。只需克隆仓库并运行 `compose.py` 即可创建自定义容器。
注意：创建的容器版本将取决于克隆的分支。例如，应该使用分支 [r24.09](https://github.com/triton-inference-server/server/tree/r24.09) 来创建基于 NGC 24.09 Triton 发布版的镜像。

`compose.py` 提供了 `--backend` 和 `--repoagent` 选项，允许您指定要包含在自定义镜像中的后端和仓库代理。例如，以下命令创建一个新的 docker 镜像，其中只包含 PyTorch 和 TensorFlow 后端以及 checksum 仓库代理。

示例：
```
python3 compose.py --backend pytorch --backend tensorflow --repoagent checksum
```
将在本地提供一个 `tritonserver` 容器。您可以使用以下命令访问该容器：
```
$ docker run -it tritonserver:latest
```

注意：如果在 `r21.08` 及更早版本上运行 `compose.py`，生成的容器将安装 DCGM 版本 2.2.3。这可能会导致 GPU 统计报告行为的差异。

### 构建特定版本的 Triton

`compose.py` 需要两个容器：一个作为构建基础的 `min` 容器和一个用于提取组件的 `full` 容器。`min` 和 `full` 容器的版本由 Triton `compose.py` 所在的分支决定。
例如，在 [r24.09](https://github.com/triton-inference-server/server/tree/r24.09) 分支上运行
```
python3 compose.py --backend pytorch --repoagent checksum
```
会拉取：
- `min` 容器 `nvcr.io/nvidia/tritonserver:24.09-py3-min`
- `full` 容器 `nvcr.io/nvidia/tritonserver:24.09-py3`

另外，用户可以通过以下任一方式从任何分支指定要拉取的 Triton 容器版本：
1. 在分支中添加标志 `--container-version <container version>`
```
python3 compose.py --backend pytorch --repoagent checksum --container-version 24.09
```
2. 指定 `--image min,<min container image name> --image full,<full container image name>`。
   用户负责指定兼容的 `min` 和 `full` 容器。
```
python3 compose.py --backend pytorch --repoagent checksum --image min,nvcr.io/nvidia/tritonserver:24.09-py3-min --image full,nvcr.io/nvidia/tritonserver:24.09-py3
```
方法 1 和 2 将生成相同的组合容器。此外，当同时指定时，`--image` 标志会覆盖 `--container-version` 标志。