<!--
# Copyright (c) 2020-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 序列扩展

本文档描述了 Triton 的序列扩展。序列扩展允许 Triton 支持有状态的模型，这些模型需要处理一系列相关的推理请求。

推理请求可以通过在请求中使用 "sequence_id" 参数来指定它是序列的一部分，并使用 "sequence_start" 和 "sequence_end" 参数来指示序列的开始和结束。

由于支持此扩展，Triton 在其服务器元数据的扩展字段中报告 "sequence"。如果 "sequence_id" 参数支持字符串类型，Triton 可能还会在服务器元数据的扩展字段中报告 "sequence(string_id)"。

- "sequence_id"：一个字符串或 uint64 值，用于标识请求所属的序列。属于同一序列的所有推理请求必须使用相同的序列 ID。序列 ID 为 0 或 "" 表示该推理请求不属于任何序列。

- "sequence_start"：布尔值，如果在请求中设置为 true，表示该请求是序列中的第一个请求。如果未设置或设置为 false，则该请求不是序列中的第一个。如果设置了此参数，"sequence_id" 参数必须设置为非零值或非空字符串值。

- "sequence_end"：布尔值，如果在请求中设置为 true，表示该请求是序列中的最后一个请求。如果未设置或设置为 false，则该请求不是序列中的最后一个。如果设置了此参数，"sequence_id" 参数必须设置为非零值或非空字符串值。

## HTTP/REST

以下示例展示了如何将请求标记为序列的一部分。在这种情况下，未使用 sequence_start 和 sequence_end 参数，这意味着该请求既不是序列的开始也不是结束。

```
POST /v2/models/mymodel/infer HTTP/1.1
Host: localhost:8000
Content-Type: application/json
Content-Length: <xx>
{
  "parameters" : { "sequence_id" : 42 }
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

下面的示例使用 v4 UUID 字符串作为 "sequence_id" 参数的值。

```
POST /v2/models/mymodel/infer HTTP/1.1
Host: localhost:8000
Content-Type: application/json
Content-Length: <xx>
{
  "parameters" : { "sequence_id" : "e333c95a-07fc-42d2-ab16-033b1a566ed5" }
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

除了支持上述序列参数外，GRPC API 还添加了推理 API 的流式版本，允许通过同一个 GRPC 流发送一系列推理请求。对于指定 sequence_id 的请求，不需要使用此流式 API，也可以用于不指定 sequence_id 的请求。ModelInferRequest 与 ModelInfer API 相同。ModelStreamInferResponse 消息如下所示。

```
service GRPCInferenceService
{
  …

  // 使用 GRPC 流执行特定模型的推理。
  rpc ModelStreamInfer(stream ModelInferRequest) returns (stream ModelStreamInferResponse) {}
}

// ModelStreamInfer 的响应消息。
message ModelStreamInferResponse
{
  // 描述错误的消息。空消息表示推理成功，没有错误。
  String error_message = 1;

  // 保存请求的结果。
  ModelInferResponse infer_response = 2;
}
```