<!--
# Copyright 2022-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 解耦后端和模型

Triton 可以支持[后端](https://github.com/triton-inference-server/backend)和模型为一个请求发送多个响应或零个响应。解耦的模型/后端还可以相对于请求批次执行的顺序发送无序的响应。这允许后端在认为合适的时候发送响应。这在自动语音识别（ASR）中特别有用。具有大量响应的请求不会阻塞其他请求的响应传递。

## 开发解耦后端/模型

### C++ 后端

请仔细阅读 [Triton 后端 API](https://github.com/triton-inference-server/backend/blob/main/README.md#triton-backend-api)、[推理请求和响应](https://github.com/triton-inference-server/backend/blob/main/README.md#inference-requests-and-responses)和[解耦响应](https://github.com/triton-inference-server/backend/blob/main/README.md#decoupled-responses)。[repeat 后端](https://github.com/triton-inference-server/repeat_backend)和 [square 后端](https://github.com/triton-inference-server/square_backend)演示了如何使用 Triton 后端 API 实现解耦后端。该示例旨在展示 Triton API 的灵活性，在任何情况下都不应在生产环境中使用。此示例可以同时处理多个请求批次，而无需增加[实例数量](model_configuration.md#instance-groups)。在实际部署中，后端不应允许调用线程从 TRITONBACKEND_ModelInstanceExecute 返回，直到该实例准备好处理另一组请求。如果设计不当，后端很容易被过度订阅。这也可能导致[动态批处理](model_configuration.md#dynamic-batcher)等功能利用不足，因为它会导致急切批处理。

### 使用 Python 后端的 Python 模型

请仔细阅读 [Python 后端](https://github.com/triton-inference-server/python_backend)，特别是 [`execute`](https://github.com/triton-inference-server/python_backend#execute)。

[解耦示例](https://github.com/triton-inference-server/python_backend/tree/main/examples/decoupled)演示了如何使用解耦 API 实现解耦 Python 模型。如示例中所述，这些示例旨在展示解耦 API 的灵活性，在任何情况下都不应在生产环境中使用。

## 部署解耦模型

必须在提供的[模型配置](model_configuration.md)文件中设置[解耦模型事务策略](model_configuration.md#decoupled)。Triton 需要此信息来启用解耦模型所需的特殊处理。在没有此配置设置的情况下部署解耦模型将在运行时抛出错误。

## 在解耦模型上运行推理

[推理协议和 API](../customization_guide/inference_protocols.md)描述了客户端与服务器通信和运行推理的各种方式。对于解耦模型，Triton 的 HTTP 端点不能用于运行推理，因为它仅支持每个请求一个响应。即使是 GRPC 端点中的标准 ModelInfer RPC 也不支持解耦响应。为了在解耦模型上运行推理，客户端必须使用双向流式 RPC。有关更多详细信息，请参见[此处](https://github.com/triton-inference-server/common/blob/main/protobuf/grpc_service.proto)。[decoupled_test.py](../../qa/L0_decoupled/decoupled_test.py)演示了如何使用 gRPC 流式处理来推理解耦模型。

如果使用 [Triton 的进程内 C API](../customization_guide/inference_protocols.md#in-process-triton-server-api)，您的应用程序应该意识到您通过 `TRITONSERVER_InferenceRequestSetResponseCallback` 注册的回调函数可以被调用任意次数，每次都会有一个新的响应。您可以查看 [grpc_server.cc](https://github.com/triton-inference-server/server/blob/main/src/grpc/grpc_server.cc)

### 了解解耦推理请求何时完成

当从模型/后端收到包含 `TRITONSERVER_RESPONSE_COMPLETE_FINAL` 标志的响应时，推理请求被认为已完成。

1. 使用流式 GRPC 的客户端应用程序可以通过检查响应参数中的 `"triton_final_response"` 参数来访问此信息。根据模型/后端的设计，解耦模型可能不会为每个请求发送响应。在这些后端不发送响应的情况下，流式 GRPC 客户端可以选择为每个请求接收一个空的最终响应。默认情况下，不发送空的最终响应以节省网络流量。

   ```python
   # 流式 GRPC 客户端选择加入的示例
   client.async_stream_infer(
     ...,
     enable_empty_final_response=True
   )
   ```

2. 使用 C API 的客户端应用程序可以在其响应处理/回调逻辑中直接检查 `TRITONSERVER_RESPONSE_COMPLETE_FINAL` 标志。

[decoupled_test.py](../../qa/L0_decoupled/decoupled_test.py)演示了通过流式 GRPC Python 客户端 API 选择加入的示例，并通过 `"triton_final_response"` 响应参数以编程方式识别何时收到最终响应。