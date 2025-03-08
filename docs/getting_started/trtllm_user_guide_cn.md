<!--
# Copyright 2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# TensorRT-LLM 用户指南

## 什么是 TensorRT-LLM

[TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM)
(TRT-LLM) 是一个开源库，旨在加速和优化 NVIDIA GPU 上大型语言模型 (LLM) 的推理性能。TRT-LLM
为用户提供了易于使用的 Python API 来构建 LLM 的 TensorRT 引擎，集成了最先进的优化技术，以确保在
NVIDIA GPU 上进行高效推理。

## 如何通过 TensorRT-LLM 后端使用 Triton Server 运行 TRT-LLM 模型

[TensorRT-LLM Backend](https://github.com/triton-inference-server/tensorrtllm_backend)
让您可以使用 Triton Inference Server 来部署 TensorRT-LLM 模型。查看 TensorRT-LLM Backend 仓库中的
[入门指南](https://github.com/triton-inference-server/tensorrtllm_backend?tab=readme-ov-file#getting-started)
部分，了解如何使用
[NGC Triton TRT-LLM 容器](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tritonserver)
为您的 LLM 模型准备引擎并使用 Triton 进行部署。

## 如何使用自定义 TRT-LLM 模型

所有支持的模型都可以在 TRT-LLM 仓库的
[examples](https://github.com/NVIDIA/TensorRT-LLM/tree/main/examples) 文件夹中找到。
按照示例将您的模型转换为 TensorRT 引擎。

构建引擎后，为 Triton [准备模型仓库](https://github.com/triton-inference-server/tensorrtllm_backend?tab=readme-ov-file#prepare-the-model-repository)
并[修改模型配置](https://github.com/triton-inference-server/tensorrtllm_backend?tab=readme-ov-file#modify-the-model-configuration)。

在模型配置文件中只需要设置*必需参数*。您可以根据需要自由修改可选参数。要了解更多关于
参数、模型输入和输出的信息，请参阅
[模型配置文档](ttps://github.com/triton-inference-server/tensorrtllm_backend/blob/main/docs/model_config.md)。

## 高级配置选项和部署策略

探索高级配置选项和部署策略，以优化和有效运行您的 TRT-LLM 模型与 Triton：

- [模型部署](https://github.com/triton-inference-server/tensorrtllm_backend/tree/main?tab=readme-ov-file#model-deployment)：在各种环境中高效部署和管理模型的技术。
- [多实例 GPU (MIG) 支持](https://github.com/triton-inference-server/tensorrtllm_backend/tree/main?tab=readme-ov-file#mig-support)：使用 MIG 运行 Triton 和 TRT-LLM 模型以优化 GPU 资源管理。
- [调度](https://github.com/triton-inference-server/tensorrtllm_backend/tree/main?tab=readme-ov-file#scheduling)：配置调度策略以控制请求的管理和执行方式。
- [键值缓存](https://github.com/triton-inference-server/tensorrtllm_backend/tree/main?tab=readme-ov-file#key-value-cache)：利用 KV 缓存和 KV 缓存重用来优化内存使用并提高性能。
- [解码](https://github.com/triton-inference-server/tensorrtllm_backend/tree/main?tab=readme-ov-file#decoding)：生成文本的高级方法，包括 top-k、top-p、top-k top-p、束搜索、Medusa 和推测性解码。
- [分块上下文](https://github.com/triton-inference-server/tensorrtllm_backend/tree/main?tab=readme-ov-file#chunked-context)：在生成阶段将上下文分成几个块并进行批处理，以提高整体吞吐量。
- [量化](https://github.com/triton-inference-server/tensorrtllm_backend/tree/main?tab=readme-ov-file#quantization)：应用量化技术以减小模型大小并提高推理速度。
- [LoRa (低秩适应)](https://github.com/triton-inference-server/tensorrtllm_backend/tree/main?tab=readme-ov-file#lora)：使用 LoRa 进行高效的模型微调和适应。

## 教程

请务必查看
[教程](https://github.com/triton-inference-server/tutorials)仓库，了解更多关于使用 Triton Server
和 TensorRT-LLM 部署流行 LLM 模型的指南，以及如何在 Kubernetes 上部署它们。

## 基准测试

[GenAI-Perf](https://github.com/triton-inference-server/perf_analyzer/tree/main/genai-perf)
是一个命令行工具，用于测量由 Triton Inference Server 部署的 LLM 的吞吐量和延迟。查看
[快速入门](https://github.com/triton-inference-server/perf_analyzer/tree/main/genai-perf#quick-start)
了解如何使用 GenAI-Perf 对您的 LLM 模型进行基准测试。

## 性能最佳实践

查看
[性能最佳实践指南](https://nvidia.github.io/TensorRT-LLM/performance/perf-best-practices.html)
了解如何优化您的 TensorRT-LLM 模型以获得更好的性能。

## 指标

Triton Server 提供
[指标](https://github.com/triton-inference-server/server/blob/main/docs/user_guide/metrics.md)
来显示 GPU 和请求统计信息。
查看 TensorRT-LLM Backend 仓库中的
[Triton 指标](https://github.com/triton-inference-server/tensorrtllm_backend?tab=readme-ov-file#triton-metrics)
部分，了解如何查询 Triton 指标端点以获取 TRT-LLM 统计信息。

## 提问或报告问题

找不到您要找的内容，或者有问题或疑问？请随时在 GitHub 问题页面上提问或报告问题：

- [TensorRT-LLM](https://github.com/NVIDIA/TensorRT-LLM/issues)
- [TensorRT-LLM Backend](https://github.com/triton-inference-server/tensorrtllm_backend/issues)
- [Triton Inference Server](https://github.com/triton-inference-server/server/issues)