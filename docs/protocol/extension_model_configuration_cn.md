<!--
# Copyright (c) 2020-2023, NVIDIA CORPORATION. All rights reserved.
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

# 模型配置扩展

本文档描述了 Triton 的模型配置扩展。模型配置扩展允许 Triton 返回服务器特定的信息。由于支持此扩展，Triton 在其服务器元数据的扩展字段中报告 "model_configuration"。

## HTTP/REST

在本文档中显示的所有 JSON 模式中，`$number`、`$string`、`$boolean`、`$object` 和 `$array` 指的是基本 JSON 类型。#optional 表示可选的 JSON 字段。

Triton 在以下 URL 暴露模型配置端点。URL 中的版本部分是可选的；如果未提供，Triton 将返回该模型最高版本号的模型配置。

```
GET v2/models/${MODEL_NAME}[/versions/${MODEL_VERSION}]/config
```

模型配置请求通过 HTTP GET 请求发送到模型配置端点。成功的模型配置请求由 200 HTTP 状态码表示。模型配置响应对象（标识为 `$model_configuration_response`）在每个成功请求的 HTTP 正文中返回。

```
$model_configuration_response =
{
  # configuration JSON
}
```

响应内容将是模型配置的 JSON 表示，该配置由 [model_config.proto 中的 ModelConfig 消息](https://github.com/triton-inference-server/common/blob/main/protobuf/model_config.proto)描述。

失败的模型配置请求必须由 HTTP 错误状态（通常为 400）表示。HTTP 正文必须包含 `$model_configuration_error_response` 对象。

```
$model_configuration_error_response =
{
  "error": <错误消息字符串>
}
```

- "error"：错误的描述性消息。

## GRPC

服务的 GRPC 定义为：

```
service GRPCInferenceService
{
  …

  // 获取模型配置
  rpc ModelConfig(ModelConfigRequest) returns (ModelConfigResponse) {}
}
```

错误由请求返回的 google.rpc.Status 表示。OK 代码表示成功，其他代码表示失败。ModelConfig 的请求和响应消息为：

```
message ModelConfigRequest
```