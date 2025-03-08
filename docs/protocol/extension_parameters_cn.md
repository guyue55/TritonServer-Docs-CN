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

# 参数扩展

本文档描述了 Triton 的参数扩展。参数扩展允许推理请求提供无法作为输入提供的自定义参数。由于支持此扩展，Triton 在其服务器元数据的扩展字段中报告 "parameters"。此扩展使用 KServe 协议中的可选 "parameters" 字段，详见 [HTTP](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#inference-request-json-object) 和 [GRPC](https://kserve.github.io/website/0.10/modelserving/data_plane/v2_protocol/#parameters)。

以下参数为 Triton 保留使用，不应用作自定义参数：

- sequence_id
- priority
- timeout
- sequence_start
- sequence_end
- headers
- 所有以 `"triton_"` 前缀开头的键。当前使用的一些示例：
  - `"triton_enable_empty_final_response"` 请求参数
  - `"triton_final_response"` 响应参数

在使用 GRPC 和 HTTP 端点时，您需要确保不使用保留参数列表以避免意外行为。保留参数在 Triton C-API 中不可访问。

## HTTP/REST

以下示例展示了请求如何包含自定义参数。

```
POST /v2/models/mymodel/infer HTTP/1.1
Host: localhost:8000
Content-Type: application/json
Content-Length: <xx>
{
  "parameters" : { "my_custom_parameter" : 42 }
  "inputs" : [
    {
      "name" : "input0",
      "shape" : [ 2, 2 ],
      "datatype" : "UINT32",
      "data" : [ 1, 2, 3, 4 ]
    }
  ],
  "outputs" : [
    {
      "name" : "output0",
    }
  ]
}
```

## GRPC

ModelInferRequest 消息中的 `parameters` 字段可用于发送自定义参数。

## 将 HTTP/GRPC 头作为参数转发

Triton 可以将 HTTP/GRPC 头作为推理请求参数转发。通过在 `--http-header-forward-pattern` 和 `--grpc-header-forward-pattern` 中指定正则表达式，Triton 将把与正则表达式匹配的头添加为请求参数。所有转发的头都将作为字符串值的参数添加。例如，要从 HTTP 和 GRPC 转发所有以 'PREFIX_' 开头的头，您应该在 `tritonserver` 命令中添加 `--http-header-forward-pattern PREFIX_.* --grpc-header-forward-pattern PREFIX_.*`。