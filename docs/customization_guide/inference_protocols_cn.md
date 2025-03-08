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

# 推理协议和 API

客户端可以使用 [HTTP/REST 协议](#httprest-and-grpc-protocols)、[GRPC 协议](#httprest-and-grpc-protocols)，或者通过[进程内 C API](#in-process-triton-server-api)及其[C++ 封装](https://github.com/triton-inference-server/developer_tools/tree/main/server)与 Triton 通信。

## HTTP/REST 和 GRPC 协议

Triton 基于 [KServe 项目](https://github.com/kserve)提出的[标准推理协议](https://github.com/kserve/kserve/tree/master/docs/predict-api/v2)提供 HTTP/REST 和 GRPC 端点。为了完全启用所有功能，Triton 还实现了 KServe 推理协议的 [HTTP/REST 和 GRPC 扩展](https://github.com/triton-inference-server/server/tree/main/docs/protocol)。GRPC 协议还提供了推理 RPC 的双向流式版本，允许通过 GRPC 流发送一系列推理请求/响应。我们通常建议使用一元版本进行推理请求。只有在情况需要时才应使用流式版本。一些这样的用例可以是：

* 假设在负载均衡器后面运行多个 Triton 服务器实例的系统。如果一系列推理请求需要命中同一个 Triton 服务器实例，GRPC 流将在整个生命周期内保持单个连接，从而确保请求被传递到同一个 Triton 实例。
* 如果需要在网络上保持请求/响应的顺序，GRPC 流将确保服务器按照从客户端发送的相同顺序接收请求。

HTTP/REST 和 GRPC 协议还提供了检查服务器和模型健康状况、元数据和统计信息的端点。其他端点允许模型加载和卸载，以及推理。有关详细信息，请参阅 KServe 和扩展文档。

### HTTP 选项
Triton 为通过 HTTP 协议进行的服务器-客户端网络事务提供以下配置选项。

#### 压缩

Triton 允许通过其客户端对 HTTP 上的请求/响应进行线上压缩。有关详细信息，请参阅 [HTTP 压缩](https://github.com/triton-inference-server/client/tree/main#compression)。

### GRPC 选项
Triton 公开了各种 GRPC 参数，用于配置服务器-客户端网络事务。有关这些选项的使用，请参阅 `tritonserver --help` 的输出。

#### SSL/TLS

这些选项可用于配置安全通信通道。服务器端选项包括：

* `--grpc-use-ssl`
* `--grpc-use-ssl-mutual`
* `--grpc-server-cert`
* `--grpc-server-key`
* `--grpc-root-cert`

有关客户端文档，请参阅[客户端 GRPC SSL/TLS](https://github.com/triton-inference-server/client/tree/main#ssltls)

有关 gRPC 身份验证概述的更多详细信息，请参阅[此处](https://grpc.io/docs/guides/auth/)。

#### 压缩

Triton 通过在服务器端公开以下选项，允许对请求/响应消息进行线上压缩：

* `--grpc-infer-response-compression-level`

有关客户端文档，请参阅[客户端 GRPC 压缩](https://github.com/triton-inference-server/client/tree/main#compression-1)

压缩可用于减少服务器-客户端通信中使用的带宽。有关更多详细信息，请参阅 [gRPC 压缩](https://grpc.github.io/grpc/core/md_doc_compression.html)。