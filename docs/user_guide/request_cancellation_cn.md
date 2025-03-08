<!--
# Copyright (c) 2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 请求取消

从 r23.10 开始，Triton 支持处理来自 gRPC 客户端或 C API 用户的请求取消。对于自动生成的大型语言模型等长时间运行的推理请求可能会运行不确定的时间或不确定的步骤数。此外，客户端可能会作为序列或请求流的一部分将大量请求排队，后来确定不再需要结果。继续处理不再需要结果的请求会显著影响服务器资源。

## 发出请求取消

### 进程内 C API

[进程内 Triton 服务器 C API](../customization_guide/inference_protocols.md#in-process-triton-server-api) 已经增强了 `TRITONSERVER_InferenceRequestCancel` 和 `TRITONSERVER_InferenceRequestIsCancelled`，分别用于发出取消和查询是否对正在进行的请求发出了取消。在 [tritonserver.h](https://github.com/triton-inference-server/core/blob/main/include/triton/core/tritonserver.h) 中阅读有关 API 的更多信息。

### gRPC 端点

此外，[gRPC 端点](../customization_guide/inference_protocols.md#httprest-and-grpc-protocols)现在可以检测来自客户端的取消并尝试终止请求。目前，只有 gRPC python 客户端支持向服务器端点发出请求取消。有关如何从客户端发出请求的更多详细信息，请参见[请求取消](https://github.com/triton-inference-server/client#request-cancellation)。有关更详细的信息，请参见 gRPC 关于 RPC [取消](https://grpc.io/docs/guides/cancellation/)的指南。

## 在 Triton Core 中的处理

Triton core 在使用[动态](./model_configuration.md#dynamic-batcher)或[序列](./model_configuration.md#sequence-batcher)批处理时，在一些关键点检查已被取消的请求。在每个[集成](./model_configuration.md#ensemble-scheduler)步骤之间也会执行检查，如果请求被取消，则终止进一步处理。

在检测到已取消的请求时，Triton core 会响应 CANCELLED 状态。如果在使用[序列批处理](./model_configuration.md#sequence-batcher)时取消了请求，则同一序列中的所有待处理请求也将被取消。序列由具有相同序列 ID 的请求表示。

**注意**：目前，一旦请求被转发到[速率限制器](./rate_limiter.md)，Triton core 就不会检测请求的取消状态。改进 Triton core 内的请求取消检测和处理仍在进行中。

## 在后端中的处理

收到请求取消后，Triton 会尽最大努力在各个点终止请求。但是，一旦请求被交给后端执行，就由各个后端来检测和处理请求终止。
目前，以下后端支持提前终止：
- [TensorRT-LLM 后端](https://github.com/triton-inference-server/tensorrtllm_backend)
- [vLLM 后端](https://github.com/triton-inference-server/vllm_backend)
- [Python 后端](https://github.com/triton-inference-server/python_backend)

Python 后端是一个特殊情况，我们公开了用于检测请求取消状态的 API，但由 `model.py` 开发人员决定是否检测请求是否被取消并终止进一步执行。

**对于后端开发人员**：后端 API 也已经增强，以让后端检测从 Triton core 收到的请求是否已被取消。有关更多详细信息，请参见 [tritonbackend.h](https://github.com/triton-inference-server/core/blob/main/include/triton/core/tritonbackend.h) 中的 `TRITONBACKEND_RequestIsCancelled` 和 `TRITONBACKEND_ResponseFactoryIsCancelled`。后端在检测到请求取消时可以停止进一步处理。
在 Python 后端后面运行的 Python 模型也可以查询请求和 response_sender 的取消状态。有关更多详细信息，请参见 python 后端文档中的[此](https://github.com/triton-inference-server/python_backend#request-cancellation-handling)部分。