<!--
# Copyright (c) 2021-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 在 Jetson 上将 Triton 推理服务器作为共享库使用

## 概述
本项目演示了如何使用 Triton 推理服务器作为共享库运行 C API 应用程序。我们还展示了如何在 Jetson 上构建和执行此类应用程序。

### 前提条件

* JetPack >= 4.6
* OpenCV >= 4.1.1
* TensorRT >= 8.0.1.6

### 安装

请按照 GitHub 发布页面的安装说明进行操作（[https://github.com/triton-inference-server/server/releases/](https://github.com/triton-inference-server/server/releases/))。

在我们的示例中，我们将下载的发布目录内容放在 `/opt/tritonserver` 下。

## 第 1 部分. 并发推理和动态批处理

位于 [concurrency_and_dynamic_batching](concurrency_and_dynamic_batching/README.md) 下的示例旨在演示 Triton 推理服务器的重要特性，如并发模型执行和动态批处理。为了实现这一点，我们使用 C API 和 Triton 推理服务器作为共享库实现了一个人员检测应用程序。

## 第 2 部分. 使用 perf_analyzer 分析模型性能

要在 Jetson 上分析模型性能，我们使用 [perf_analyzer](https://github.com/triton-inference-server/perf_analyzer/blob/main/README.md) 工具。`perf_analyzer` 包含在发布的 tar 文件中，或者可以从源代码编译。

从仓库的这个目录执行以下命令来评估模型性能：

```shell
./perf_analyzer -m peoplenet -b 2 --service-kind=triton_c_api --model-repo=$(pwd)/concurrency_and_dynamic_batching/trtis_model_repo_sample_1 --triton-server-directory=/opt/tritonserver --concurrency-range 1:6 -f perf_c_api.csv
```

在上面的示例中，我们将结果保存为 `.csv` 文件。要可视化这些结果，请按照[此处](https://github.com/triton-inference-server/perf_analyzer/blob/main/README.md)描述的步骤操作。