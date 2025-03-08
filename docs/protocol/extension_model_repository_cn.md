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

# 模型仓库扩展

本文档描述了 Triton 的模型仓库扩展。模型仓库扩展允许客户端查询和控制 Triton 正在服务的一个或多个模型仓库。由于支持此扩展，Triton 在服务器元数据的扩展字段中报告 "model_repository"。此扩展有一个可选组件，如下所述，允许卸载 API 指定 "unload_dependents" 参数。支持此可选组件的 Triton 版本还将在服务器元数据的扩展字段中报告 "model_repository(unload_dependents)"。

## HTTP/REST

在本文档中显示的所有 JSON 模式中，`$number`、`$string`、`$boolean`、`$object` 和 `$array` 指的是基本 JSON 类型。`#optional` 表示可选的 JSON 字段。

模型仓库扩展需要索引、加载和卸载 API。Triton 在以下 URL 暴露这些端点。

```
POST v2/repository/index

POST v2/repository/models/${MODEL_NAME}/load

POST v2/repository/models/${MODEL_NAME}/unload
```

### 索引

索引 API 返回有关模型仓库中每个可用模型的信息，即使该模型当前未加载到 Triton 中。索引 API 提供了一种方法来确定哪些模型可以通过加载 API 加载。模型仓库索引请求通过 HTTP POST 发送到索引端点。在相应的响应中，HTTP 正文包含 JSON 响应。

索引请求对象（标识为 `$repository_index_request`）必须在 POST 请求的 HTTP 正文中提供。

```
$repository_index_request =
{
  "ready" : $boolean #optional,
}
```

- "ready"：可选，默认为 false。如果为 true，则仅返回准备好进行推理的模型。

成功的索引请求由 200 HTTP 状态码表示。响应对象（标识为 `$repository_index_response`）在每个成功请求的 HTTP 正文中返回。

```
$repository_index_response =
[
  {
    "name" : $string,
    "version" : $string #optional,
    "state" : $string,
    "reason" : $string
  },
  …
]
```

- "name"：模型的名称。
- "version"：模型的版本。
- "state"：模型的状态。
- "reason"：模型处于当前状态的原因（如果有）。