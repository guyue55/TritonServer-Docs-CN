<!--
# Copyright (c) 2019-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 优化

Triton 推理服务器具有许多功能，您可以使用这些功能来降低延迟并提高模型的吞吐量。本节讨论这些功能，并演示如何使用它们来提高模型的性能。作为先决条件，您应该按照[快速入门](../getting_started/quickstart.md)指南来运行 Triton 和客户端示例，并使用示例模型仓库。

本节重点介绍理解单个模型的延迟和吞吐量权衡。[模型分析器](model_analyzer.md)部分描述了一个工具，可以帮助您了解模型的 GPU 内存使用情况，以便您可以决定如何在单个 GPU 上最佳地运行多个模型。

除非您已经有一个适合测量模型在 Triton 上性能的客户端应用程序，否则您应该熟悉[性能分析器](https://github.com/triton-inference-server/perf_analyzer/blob/main/README.md)。性能分析器是优化模型性能的重要工具。

作为演示优化功能和选项的运行示例，我们将使用一个 TensorFlow Inception 模型，您可以通过按照[快速入门](../getting_started/quickstart.md)获取该模型。作为基准，我们使用 perf_analyzer 来确定使用[不启用任何性能功能的基本模型配置](../examples/model_repository/inception_graphdef/config.pbtxt)的模型性能。

```
$ perf_analyzer -m inception_graphdef --percentile=95 --concurrency-range 1:4
...
Inferences/Second vs. Client p95 Batch Latency
Concurrency: 1, throughput: 62.6 infer/sec, latency 21371 usec
Concurrency: 2, throughput: 73.2 infer/sec, latency 34381 usec
Concurrency: 3, throughput: 73.2 infer/sec, latency 50298 usec
Concurrency: 4, throughput: 73.4 infer/sec, latency 65569 usec
```

结果显示，我们未优化的模型配置每秒可以处理约 73 次推理。注意，从一个并发请求增加到两个并发请求时吞吐量显著提高，然后吞吐量趋于平稳。使用一个并发请求时，Triton 在将响应返回给客户端并在服务器收到下一个请求的时间内处于空闲状态。使用两个并发请求时吞吐量增加，因为 Triton 可以将一个请求的处理与另一个请求的通信重叠。由于我们在与 Triton 相同的系统上运行 perf_analyzer，两个请求足以完全隐藏通信延迟。

## 优化设置

对于大多数模型来说，提供最大性能改进的 Triton 功能是[动态批处理](model_configuration.md#dynamic-batcher)。[本示例](https://github.com/triton-inference-server/tutorials/tree/main/Conceptual_Guide/Part_2-improving_resource_utilization#dynamic-batching--concurrent-model-execution)更详细地阐述了概念细节。如果您的模型不支持批处理，则可以跳到[模型实例](#model-instances)部分。