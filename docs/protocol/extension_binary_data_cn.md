<!--
# Copyright 2020-2022, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 二进制张量数据扩展

本文档描述了 Triton 的二进制张量数据扩展。二进制张量数据扩展允许 Triton 支持在 HTTP/REST 请求体中以二进制格式表示的张量数据。由于支持此扩展，Triton 在其服务器元数据的 extensions 字段中报告 "binary_tensor_data"。

## 二进制张量请求

以二进制数据表示的张量数据按小端字节序、行优先顺序组织，元素之间没有步长或填充。所有张量数据类型都可以用数据类型的原生大小表示为二进制数据。对于 BOOL 类型，元素 true 是值为 1 的单个字节，false 是值为 0 的单个字节。对于 BYTES 类型，一个元素由一个 4 字节无符号整数表示长度，后跟实际字节。张量的二进制数据在 JSON 对象之后在 HTTP 请求体中传递（参见示例）。

二进制张量数据扩展使用参数来指示输入或输出张量是以二进制数据形式通信的。第一个参数用在 `$request_input` 和 `$response_output` 中，表示输入或输出张量是以二进制数据形式通信的：

- "binary_data_size"：int64 参数，表示张量二进制数据的大小（以字节为单位）。

第二个参数用在 `$request_output` 中，表示输出应该从 Triton 返回为二进制数据：

- "binary_data"：bool 参数，如果为 true 则输出应该返回为二进制数据，如果为 false（或未给出）则张量应该返回为 JSON。

第三个参数用在 $inference_request 中，表示所有输出都应该从 Triton 返回为二进制数据，除非被特定输出的 "binary_data" 覆盖：

- "binary_data_output"：bool 参数，如果为 true 则所有输出都应该返回为二进制数据，如果为 false（或未给出）则输出应该返回为 JSON。如果在输出上指定了 "binary_data"，则它会覆盖此设置。

当一个或多个张量以二进制数据形式通信时，请求或响应的 HTTP 请求体将包含 JSON 推理请求或响应对象，后跟按照 JSON 中指定的输入或输出张量顺序排列的二进制张量数据。如果请求或响应中存在任何二进制数据，则必须提供 Inference-Header-Content-Length 头部来给出 JSON 对象的长度，而 Content-Length 继续给出完整的请求体长度（这是 HTTP 要求的）。

### 示例

对于以下请求，输入张量作为二进制数据发送，并且输出张量必须作为二进制数据返回，因为这是请求的内容。另外请注意，二进制数据的总大小为 19 字节，该大小必须反映在内容长度头部中。

```
POST /v2/models/mymodel/infer HTTP/1.1
Host: localhost:8000
Content-Type: application/octet-stream
Inference-Header-Content-Length: <xx>
Content-Length: <xx+19>
{
  "model_name" : "mymodel",
  "inputs" : [
    {
      "name" : "input0",
      "shape" : [ 2, 2 ],
      "datatype" : "UINT32",