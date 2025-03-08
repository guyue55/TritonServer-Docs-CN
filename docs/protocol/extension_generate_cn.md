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

# 生成扩展

> [!NOTE]
> 生成扩展是*临时性的*，在未来的版本中可能会发生变化。

本文档描述了 Triton 的生成扩展。生成扩展提供了一个简单的面向文本的端点模式，用于与大型语言模型（LLMs）交互。生成端点是 HTTP/REST 前端特有的。

## HTTP/REST

在本文档中显示的所有 JSON 模式中，`$number`、`$string`、`$boolean`、`$object` 和 `$array` 指的是基本 JSON 类型。#optional 表示可选的 JSON 字段。

Triton 在以下 URL 暴露生成端点。客户端可以使用 HTTP POST 请求访问不同的 URL 以获得不同的响应行为，端点在成功时将返回生成结果，在失败时返回错误。

```
POST v2/models/${MODEL_NAME}[/versions/${MODEL_VERSION}]/generate

POST v2/models/${MODEL_NAME}[/versions/${MODEL_VERSION}]/generate_stream
```

### generate 与 generate_stream

两个 URL 都期望相同的请求 JSON 对象，并生成相同的 JSON 响应对象。但是，返回格式有一些差异：
* `/generate` 返回恰好 1 个响应 JSON 对象，`Content-Type` 为 `application/json`
* `/generate_stream` 可能根据推理结果返回多个响应，`Content-Type` 为 `text/event-stream; charset=utf-8`。
这些响应将作为[服务器发送事件](https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events)（SSE）发送，
其中每个响应将是 HTTP 响应体中的一个 "data" 块。在推理错误的情况下，响应将包含一个[错误 JSON 对象](#generate-response-json-error-object)。
    * 请注意，HTTP 响应代码在 SSE 的第一个响应中设置，因此如果第一个响应成功但在请求的后续响应中发生错误，
    可能会在状态码显示成功（200）的同时收到错误对象。因此，用户在通过 `/generate_stream` 生成响应时必须始终检查是否收到错误对象。
    * 如果请求在推理开始前失败，则将返回一个 JSON 错误，其 `Content-Type` 为 `application/json`，类似于来自其他端点的错误，状态码设置为错误。

### 生成请求 JSON 对象

生成请求对象（标识为 *$generate_request*）在 POST 请求的 HTTP 体中是必需的。模型名称和（可选的）版本必须在 URL 中可用。如果未提供版本，服务器可能根据其自身策略选择版本或返回错误。

    $generate_request =
    {
      "id" : $string, #optional
      "text_input" : $string,
      "parameters" : $parameters #optional
    }

* "id"：此请求的标识符。可选，但如果指定，此标识符必须在响应中返回。
* "text_input"：模型应该从中生成输出的文本输入。
* "parameters"：一个可选对象，包含零个或多个以键/值对表示的生成请求参数。有关更多信息，请参见[参数](#parameters)。