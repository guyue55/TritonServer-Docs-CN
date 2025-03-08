<!--
# Copyright 2022-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 使用 Triton 部署您的训练模型

给定一个训练好的模型，如何使用 Triton Inference Server 以最佳配置进行大规模部署？本文档将帮助您回答这个问题。

对于喜欢[概述](#overview)的人来说，以下是大多数用例的常见流程。

对于想直接开始的人，可以跳转到[端到端示例](#end-to-end-example)。

有关其他材料，请参阅 [Triton 概念指南教程](https://github.com/triton-inference-server/tutorials/tree/main/Conceptual_Guide/Part_4-inference_acceleration)。

## 概述

1. 我的模型是否与 Triton 兼容？
    - 如果您的模型属于 Triton 的[支持后端](https://github.com/triton-inference-server/backend)之一，那么我们可以按照[快速入门](../getting_started/quickstart.md)指南中的描述尝试部署模型。
    对于 ONNXRuntime、TensorFlow SavedModel 和 TensorRT 后端，可以使用 Triton 的[自动完成](model_configuration.md#auto-generated-model-configuration)功能从模型中推断出最小模型配置。
    这意味着可以提供 `config.pbtxt`，但除非您想要显式设置某些参数，否则不需要。
    此外，通过启用详细日志记录（`--log-verbose=1`），您可以在服务器日志输出中看到 Triton 内部看到的完整配置。
    对于其他后端，请参阅[最小模型配置](model_configuration.md#minimal-model-configuration)以开始使用。
    - 如果您的模型不来自支持的后端，您可以考虑使用 [Python 后端](https://github.com/triton-inference-server/python_backend)或编写[自定义 C++ 后端](https://github.com/triton-inference-server/backend/blob/main/examples/README.md)来支持您的模型。Python 后端提供了一个简单的接口，可以通过通用 Python 脚本执行请求，但性能可能不如自定义 C++ 后端。根据您的用例，Python 后端的性能可能足以换取实现的简单性。

2. 我能否在已部署的模型上运行推理？
    - 假设您能够在 Triton 上加载模型，下一步是验证我们可以运行推理请求并获取模型的基准性能基准。
    Triton 的 [Perf Analyzer](https://github.com/triton-inference-server/perf_analyzer/blob/main/README.md) 工具专门适合这个目的。以下是一个简化的输出示例：

    ```
    # 注意："my_model" 代表当前由 Triton 提供服务的模型
    $ perf_analyzer -m my_model
    ...

    Inferences/Second vs. Client Average Batch Latency
    Concurrency: 1, throughput: 482.8 infer/sec, latency 12613 usec
    ```

    - 这为我们提供了一个完整性测试，确保我们能够成功形成输入请求并通过 Triton API 接收输出响应与模型后端通信。
    - 如果 Perf Analyzer 无法发送请求，并且从错误中不清楚如何继续，那么您可能需要检查您的模型 `config.pbtxt` 输入/输出是否与模型期望的匹配。如果配置正确，请检查模型是否可以使用其原始框架直接成功运行。如果您没有自己的脚本或工具来执行此操作，[Polygraphy](https://github.com/NVIDIA/TensorRT/tree/main/tools/Polygraphy) 是一个有用的工具，可以通过各种框架对您的模型运行示例推理。目前，Polygraphy 支持 ONNXRuntime、TensorRT 和 TensorFlow 1.x。
    - "性能良好