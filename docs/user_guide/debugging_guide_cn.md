<!--
# Copyright 2023-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 调试指南
本指南介绍了在 Triton 出现异常行为或失败时的常见场景的首要故障排除步骤。以下我们将问题分为以下几类：

- **[配置](#配置问题)**：Triton 报告您的配置文件有错误。
- **[模型](#模型问题)**：您的模型无法加载或执行推理。
- 服务器：服务器崩溃或不可用。
- 客户端：客户端在向服务器发送和接收数据时失败。
- 性能：Triton 未达到最佳性能。

无论您遇到的是哪一类问题，尽可能尝试在最新的 Triton 容器中运行都是值得的。虽然我们为旧容器提供支持，但修复会合并到下一个版本中。通过检查最新版本，您可以发现这个问题是否已经解决。

您还可以搜索 [Triton 的 GitHub issues](https://github.com/triton-inference-server/server/issues) 来查看是否有人之前询问过您的问题。如果您收到了错误信息，可以使用错误中的几个关键词作为搜索词。

Triton 提供了不同类型的错误和状态，与广泛的问题相关。以下是它们的概述：

| 错误 | 定义 | 示例 |
| ----- | ---------- | ------- |
|已存在 | 当由于已存在项目而无法执行操作时返回。 | 已注册的模型无法再次注册。|
| 内部 | 当 Triton 代码内部出现意外失败时返回。 | 内存分配失败。 |
| 无效参数 | 当向函数提供无效参数时返回 | 模型配置有无效参数 |
| 未找到 | 当请求的资源无法找到时返回 | 无法找到共享库 |
| 不可用 | 当找到请求的资源但不可用时返回 | 请求的模型尚未准备好进行推理 |
| 未知 | 当错误原因未知时返回 | 不应使用此错误代码 |
| 不支持 | 当选项不受支持时返回 | 模型配置包含该后端尚不支持的参数 |

## 配置问题

在继续之前，请查看[此处](./model_configuration.md)的模型配置文档是否解决了您的问题。除此之外，找到适合您用例的示例模型配置的最佳位置是：

- 服务器的 [qa 文件夹](https://github.com/triton-inference-server/server/tree/main/qa)。您可以找到涵盖大多数功能的测试脚本，包括一些更新模型配置文件的脚本。
    - [Custom_models](https://github.com/triton-inference-server/server/tree/main/qa/custom_models)、[ensemble_models](https://github.com/triton-inference-server/server/tree/main/qa/ensemble_models) 和 [python_models](https://github.com/triton-inference-server/server/tree/main/qa/python_models) 包含了各自用例的配置示例。
    - [L0_model_config](https://github.com/triton-inference-server/server/tree/main/qa/L0_model_config) 测试了许多类型的不完整模型配置。

请注意，如果您遇到 [perf_analyzer](https://github.com/triton-inference-server/perf_analyzer/blob/main/README.md) 或 [Model Analyzer](https://github.com/triton-inference-server/model_analyzer) 的问题，请尝试直接在 Triton 上加载模型。这可以检查配置是否不正确或是否需要更新 perf_analyzer 或 Model Analyzer 选项。

## 模型问题
**步骤 1. 在 Triton 外运行模型**

如果您在加载或运行模型时遇到问题，第一步是确保您的模型可以在其框架外部运行。例如，您可以在 ONNX Runtime 中运行 ONNX 模型，在 trtexec 中运行 TensorRT 模型。如果此检查失败，则问题出在框架内部而不是 Triton 内部。

**步骤 2. 找到错误信息**

如果您收到错误信息，您可以通过搜索代码来找到它是在哪里生成的。GitHub 提供了[此处](https://docs.github.com/en/search-github/searching-on-github/searching-code)的代码搜索说明。可以在[此链接](https://github.com/search?q=org%3Atriton-inference-server&type=Code)中搜索 Triton 组织。

如果您的错误信息在 Triton 代码中只出现在一个或几个地方，您可能很快就能看出问题所在。即使不是，保存这个链接以便在寻求帮助时提供给我们也是很好的。这通常是我们首先要查找的内容。

**步骤 3. 使用调试标志构建**

下一步是使用调试标志构建。我们很遗憾不提供调试容器，所以您需要按照[构建指南](https://github.com/triton-inference-server/server/blob/main/docs/customization_guide/build.md)来构建容器，其中包括[添加调试符号的部分](https://github.com/triton-inference-server/server/blob/main/docs/build.md#building-with-debug-symbols)。完成后，您可以在容器中安装 GDB (`apt-get install gdb`)并在 GDB 中运行 Triton (`gdb --args tritonserver…`)。如果需要，您可以打开第二个终端在另一个容器中运行脚本。如果服务器段错误，您可以输入 `backtrace`，这将提供一个调用堆栈，让您知道错误是在哪里生成的。然后您应该能够追踪错误的来源。如果调试后错误仍然存在，我们需要这些信息来加快我们的工作。

高级 GDB 用户还可以检查变量值、添加断点等来找出问题的原因。

### 特定问题
**未定义符号**

这里有几个选项：
- 这通常意味着 Triton 使用的框架版本与用于创建模型的版本不匹配。检查 Triton 容器中使用的框架版本，并与用于生成模型的版本进行比较。
- 如果您正在加载后端使用的共享库，不要忘记在运行 Tritonserver 的命令之前包含 LD_PRELOAD。
    - `LD_PRELOAD=<name_of_so_file.so> tritonserver --model-repository…`
如果您自己构建了后端，这可能是一个链接错误。如果您确信后端和服务器构建正确，请仔细检查服务器是否加载了正确的后端。

## 服务器问题

通常您不应该遇到服务器本身的错误。如果服务器宕机，通常是因为在模型加载或推理过程中出现了问题，您可以使用上面的部分进行调试。使用上面的[使用调试标志构建](https://github.com/triton-inference-server/server/blob/main/docs/build.md#building-with-debug-symbols)部分来解决这类问题特别有用。但是，本节将介绍一些可能出现的特定情况。

### 无法连接到服务器

如果您在连接服务器或通过健康端点获取其健康状态时遇到问题(`curl -v localhost:8000/v2/health/ready`)，请确保您可以从运行命令的位置访问服务器运行的网络。最常见的情况是，当为客户端和服务器启动单独的 Docker 容器时，它们没有使用 [--net=host](https://docs.docker.com/network/host/) 来共享网络。

### 间歇性故障

这将是最难调试的问题之一。如果可能，您要使用调试标志构建服务器，以获取发生的具体情况的回溯。您还要记录笔记，看看这种情况发生的频率以及是否有共同的原因。服务器本身在空闲时不应该失败，所以看看是否某个特定操作（加载/卸载模型、运行模型推理等）触发了它。

### 由于单个模型导致的服务器故障

如果您希望即使在模型失败的情况下服务器也能启动，请使用 `exit-on-error=false` 选项。如果您希望即使在特定模型失败的情况下服务器健康端点也显示就绪，请使用 `--strict-readiness=false` 标志。

### 死锁

使用 `gdb` 调试死锁的一些有用步骤：
1. 使用 `$info threads` 查看哪些线程正在等待。
2. 转到一个线程：`$thread 4`。
3. 打印回溯：`$bt`。
4. 转到带有锁的帧：`$f 1`。
5. 打印被持有的互斥锁的内存：`$p *mutex`。
6. 现在您可以在 `owner` 下看到互斥锁的所有者。

## 客户端问题

对于处理不同的客户端情况，最好的资源是[客户端仓库](https://github.com/triton-inference-server/client)的示例。您可以看到用 Python、Java 和 C++ 编写的客户端，涵盖了许多常见用例的运行示例。您可以查看这些客户端的主要功能，以了解代码的流程。

我们经常收到关于客户端性能优化的问题。Triton 客户端将输入张量作为原始二进制发送。但是，GRPC 使用 protobuf，这会带来一些序列化和反序列化的开销。对于那些寻求最低延迟解决方案的人来说，C API 消除了与 GRPC/HTTP 相关的延迟。当客户端和服务器在同一系统上时，共享内存也是减少数据移动的好选择。

## 性能问题

本节介绍调试意外性能。如果您想优化性能，请参阅[优化](https://github.com/triton-inference-server/server/blob/main/docs/optimization.md)和[性能调优](https://github.com/triton-inference-server/server/blob/main/docs/performance_tuning.md)指南。

最简单的开始步骤是运行 perf_analyzer 来获取每个模型的请求生命周期、吞吐量和延迟的细分。要获得更详细的视图，您可以在运行服务器时[启用跟踪](https://github.com/triton-inference-server/server/blob/main/docs/trace.md)。这将提供精确的时间戳以深入了解发生的情况。您还可以通过使用跟踪标志为 GRPC 和 HTTP 客户端启用 perf_analyzer 的跟踪。请注意，启用跟踪可能会影响 Triton 的性能，但它可以帮助检查请求生命周期中的时间戳。

### 性能分析

下一步是使用性能分析器。我们推荐的一个分析器是 [Nsight Systems](https://developer.nvidia.com/nsight-systems) (nsys)，可以选择包括 NVIDIA Tools Extension (NVTX) 标记来分析 Triton