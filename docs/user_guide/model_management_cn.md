<!--
# Copyright 2018-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 模型管理

Triton 提供的模型管理 API 是 [HTTP/REST 和 GRPC 协议以及 C API](../customization_guide/inference_protocols.md) 的一部分。Triton 可以在三种模型控制模式之一下运行：NONE、EXPLICIT 或 POLL。模型控制模式决定了 Triton 如何处理模型仓库的变更，以及哪些协议和 API 可用。

## 模型控制模式 NONE

Triton 在启动时尝试加载模型仓库中的所有模型。Triton 无法加载的模型将被标记为 UNAVAILABLE，并且无法用于推理。

服务器运行时对模型仓库的更改将被忽略。使用[模型控制协议](../protocol/extension_model_repository.md)的模型加载和卸载请求将不会生效，并将返回错误响应。

通过在启动 Triton 时指定 `--model-control-mode=none` 来选择此模型控制模式。这是默认的模型控制模式。在 Triton 运行时更改模型仓库必须谨慎进行，具体说明请参见[修改模型仓库](#修改模型仓库)。

## 模型控制模式 EXPLICIT

在启动时，Triton 仅加载通过 `--load-model` 命令行选项明确指定的模型。要在启动时加载所有模型，请将 `--load-model=*` 作为唯一的 `--load-model` 参数。将 `--load-model=*` 与其他 `--load-model` 参数一起使用将导致错误。如果未指定 `--load-model`，则在启动时不会加载任何模型。Triton 无法加载的模型将被标记为 UNAVAILABLE，并且无法用于推理。

启动后，所有模型加载和卸载操作必须通过使用[模型控制协议](../protocol/extension_model_repository.md)明确启动。模型控制请求的响应状态表示加载或卸载操作的成功或失败。当尝试重新加载已加载的模型时，如果重新加载因任何原因失败，已加载的模型将保持不变并继续保持加载状态。如果重新加载成功，新加载的模型将替换已加载的模型，而不会导致模型的可用性出现任何损失。

通过指定 `--model-control-mode=explicit` 启用此模型控制模式。在 Triton 运行时更改模型仓库必须谨慎进行，具体说明请参见[修改模型仓库](#修改模型仓库)。

如果您在使用[模型控制协议](../protocol/extension_model_repository.md)加载和卸载模型时发现内存增长，这可能不是实际的内存泄漏，而是系统的 malloc 启发式算法导致内存无法立即释放回操作系统。要改善内存性能，您可以考虑通过设置 `LD_PRELOAD` 环境变量从 malloc 切换到 [tcmalloc](https://github.com/google/tcmalloc) 或 [jemalloc](https://github.com/jemalloc/jemalloc)，如下所示：
```
# 使用 tcmalloc
LD_PRELOAD=/usr/lib/$(uname -m)-linux-gnu/libtcmalloc.so.4:${LD_PRELOAD} tritonserver --model-repository=/models ...
```
```
# 使用 jemalloc
LD_PRELOAD=/usr/lib/$(uname -m)-linux-gnu/libjemalloc.so:${LD_PRELOAD} tritonserver --model-repository=/models ...
```
我们建议同时尝试 tcmalloc 和 jemalloc，以确定哪一个更适合您的用例，因为它们在内存分配和释放方面采用不同的策略，可能会根据工作负载表现出不同的性能。

tcmalloc 和 jemalloc 库已经预装在 Triton 容器中。但是，如果您需要安装它们，可以使用以下命令：
```
# 安装 tcmalloc
apt-get install gperf libgoogle-perftools-dev
```
```
# 安装 jemalloc
apt-get install libjemalloc-dev
```

## 模型控制模式 POLL

Triton 在启动时尝试加载模型仓库中的所有模型。Triton 无法加载的模型将被标记为 UNAVAILABLE，并且无法用于推理。

对模型仓库的更改将被检测到，Triton 将根据这些更改尝试加载和卸载模型。当尝试重新加载已加载的模型时，如果重新加载因任何原因失败，已加载的模型将保持不变并继续保持加载状态。如果重新加载成功，新加载的模型将替换已加载的模型，而不会导致模型的可用性出现任何损失。

由于 Triton 定期轮询仓库，因此可能不会立即检测到对模型仓库的更改。您可以使用 `--repository-poll-secs` 选项控制轮询间隔。可以使用控制台日志、[模型就绪协议](https://github.com/kserve/kserve/blob/master/docs/predict-api/v2/required_api.md)或[模型控制协议](../protocol/extension_model_repository.md)的索引操作来确定模型仓库更改何时生效。

**警告：Triton 轮询模型仓库的时间与您对仓库进行更改的时间之间没有同步。因此，Triton 可能会观察到部分和不完整的更改，导致意外行为。因此不建议在生产环境中使用 POLL 模式。**

使用[模型控制协议](../protocol/extension_model_repository.md)的模型加载和卸载请求将不会生效，并将返回错误响应。

通过在启动 Triton 时指定 `--model-control-mode=poll` 并将 `--repository-poll-secs` 设置为非零值来启用此模型控制模式。在 Triton 运行时更改模型仓库必须谨慎进行，具体说明请参见[修改模型仓库](#修改模型仓库)。

在 POLL 模式下，Triton 响应以下模型仓库更改：

* 可以通过添加和删除相应的版本子目录来添加和删除模型的版本。即使正在使用已删除的模型版本，Triton 也会允许正在进行的请求完成。对已删除模型版本的新请求将失败。根据模型的[版本策略](model_configuration.md#version-policy)，可用版本的更改可能会改变默认提供的模型版本。

* 可以通过删除相应的模型目录从仓库中删除现有模型。Triton 将允许对已删除模型的任何版本的正在进行的请求完成。对已删除模型的新请求将失败。

* 可以通过添加新的模型目录向仓库添加新模型。

* 可以更改[模型配置文件](model_configuration.md)（config.pbtxt），Triton 将卸载并重新加载模型以应用新的模型配置。

* 可以添加、删除或修改为表示分类的输出提供标签的标签文件，Triton 将卸载并重新加载模型以应用新的标签。如果添加或删除了标签文件，必须同时对[模型配置](model_configuration.md)中相应输出的 *label_filename* 属性进行相应的编辑。

## 修改模型仓库

模型仓库中的每个模型都[位于其自己的子目录中](model_repository.md#repository-layout)。允许对模型子目录内容进行的活动取决于 Triton 如何使用该模型。可以使用[模型元数据](../customization_guide/inference_protocols.md#inference-protocols-and-apis)或[仓库索引](../protocol/extension_model_repository.md#index) API 确定模型的状态。

* 如果模型正在加载或卸载，则不得添加、删除或修改该子目录中的任何文件或目录。

* 如果模型从未加载或已完全卸载，则可以删除整个模型子目录，或添加、删除或修改其任何内容。

* 如果模型已完全加载，则可以添加、删除或修改该子目录中的任何文件或目录；但实现模型后端的共享库除外。Triton 在模型加载时使用后端共享库，因此删除或修改它们可能会导致 Triton 崩溃。要更新模型的后端，您必须首先完全卸载模型，修改后端共享库，然后重新加载模型。在某些操作系统上，也可以简单地将现有共享库移动到模型仓库外的另一个位置，复制新的共享库，然后重新加载模型。

* 如果仅修改了 'config.pbtxt' 中的模型实例配置（即增加/减少实例数），则当在[模型控制模式 EXPLICIT](#模型控制模式-explicit)下收到加载请求或在[模型控制模式 POLL](#模型控制模式-poll)下检测到对 'config.pbtxt' 的更改时，Triton 将更新模型而不是重新加载它。
  * 新的模型配置也可以通过[加载 API](../protocol/extension_model_repository.md#load)传递给 Triton。
  * 某些文本编辑器在就地修改 'config.pbtxt' 时会在模型目录中创建交换文件。交换文件不是模型配置的一部分，因此它在模型目录中的存在可能会被检测为新文件，并在仅期望更新时导致模型完全重新加载。

* 如果序列模型被*更新*（即减少实例数），Triton 将等待正在进行的序列完成（或超时）后再删除序列背后的实例。
  * 如果减少实例数，将从空闲实例和具有正在进行的序列的实例中任意选择实例进行删除。

* 如果序列模型在有正在进行的序列时被*重新加载*（即更改模型文件），Triton 不保证来自正在进行的序列的任何剩余请求将被路由到相同的模型实例进行处理。目前，用户有责任确保在重新加载序列模型之前完成所有正在进行的序列。

## 并发加载模型

为了减少服务停机时间，Triton 在后台加载新模型，同时继续对现有模型提供推理服务。根据用例和性能要求，用于加载模型的最佳资源量可能会有所不同。Triton 提供了 `--model-load-thread-count` 选项来配置专用于加载模型的线程数，默认值为 4。

要使用 C API 设置此参数，请参阅 [tritonserver.h](https://github.com/triton-inference-server/core/blob/main/include/triton/core/tritonserver.h) 中的 `TRITONSERVER_ServerOptionsSetModelLoadThreadCount`。