<!--
# Copyright (c) 2022, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 跟踪扩展

本文档描述了 Triton 的跟踪扩展。跟踪扩展使客户端能够在 Triton 运行期间配置跟踪设置。由于支持此扩展，Triton 在其服务器元数据的扩展字段中报告 "trace"。

## HTTP/REST

在本文档中显示的所有 JSON 模式中，`$number`、`$string`、`$boolean`、`$object` 和 `$array` 指的是基本 JSON 类型。`#optional` 表示可选的 JSON 字段。

Triton 在以下 URL 暴露跟踪端点。客户端可以使用 HTTP GET 请求获取当前的跟踪设置。HTTP POST 请求将修改跟踪设置，成功时端点将返回更新后的跟踪设置，失败时将返回错误。可以提供可选的模型名称来获取或设置特定模型的跟踪设置。

```
GET v2[/models/${MODEL_NAME}]/trace/setting

POST v2[/models/${MODEL_NAME}]/trace/setting
```

### 跟踪设置响应 JSON 对象

成功的跟踪设置请求由 200 HTTP 状态码表示。响应对象（标识为 `$trace_setting_response`）在每个成功请求的 HTTP 正文中返回。

```
$trace_setting_response =
{
  $trace_setting, ...
}

$trace_setting = $string : $string | [ $string, ...]
```

每个 `$trace_setting` JSON 描述一个 "名称"/"值" 对，其中 "名称" 是跟踪设置的名称，"值" 是设置值的 `$string` 表示，或者对于某些设置是 `$string` 数组。目前定义了以下跟踪设置：

- "trace_file"：保存跟踪输出的文件。如果设置了 "log_frequency"，这将是保存跟踪输出的文件的前缀，生成的文件名为 `"${trace_file}.0"、"${trace_file}.1"、...`，详见下面的 "log_frequency" 跟踪设置。
- "trace_level"：跟踪级别。"OFF" 禁用跟踪，"TIMESTAMPS" 跟踪时间戳，"TENSORS" 跟踪张量。此值是一个字符串数组，用户可以指定多个级别来跟踪多个信息。
- "trace_rate"：跟踪采样率。该值表示每多少个请求采样一个跟踪。例如，如果跟踪率是 "1000"，则每 1000 个请求采样 1 个跟踪。
- "trace_count"：要采样的剩余跟踪数。一旦值变为 "0"，将不再为该跟踪设置采样跟踪，并且收集的跟踪将以 "log_frequency" 中描述的格式写入索引跟踪文件，无论 "log_frequency" 状态如何。如果值为 "-1"，则要采样的跟踪数将不受限制。
- "log_frequency"：Triton 将跟踪输出记录到文件的频率。如果值为 "0"，Triton 将仅在关闭时将跟踪输出记录到 `${trace_file}`。否则，当 Triton 收集到指定数量的跟踪时，将跟踪输出记录到 `${trace_file}.${idx}`。例如，如果日志频率为 "100"，当 Triton 收集到第 100 个跟踪时，它将跟踪记录到文件 `"${trace_file}.0"`，当收集到第 200 个跟踪时，它将第 101 个到第 200 个跟踪记录到文件 `"${trace_file}.1"`。注意，当更新 "trace_file" 设置时，文件索引将重置为 0。

### 跟踪设置响应 JSON 错误对象

失败的跟踪设置请求将由 HTTP 错误状态（通常为 400）表示。HTTP 正文必须包含 `$trace_setting_error_response` 对象。

```
$trace_setting_error_response =
{
  "error": $string
}
```

- "error"：错误的描述性消息。

#### 跟踪设置请求 JSON 对象

跟踪设置请求通过 HTTP POST 发送到跟踪端点。在相应的响应中，HTTP 正文包含响应 JSON。成功的请求由 200 HTTP 状态码表示。

请求对象（标识为 `$trace_setting_request`）必须在 HTTP 正文中提供。

```
$trace_setting_request =
{
  $trace_setting, ...
}
```

`$trace_setting` JSON 在[跟踪设置响应 JSON 对象](#跟踪设置响应-json-对象)中定义，只有指定的设置将被更新。除了响应 JSON 对象中提到的值外，还可以使用 JSON null 值来移除跟踪设置的规范。在这种情况下，将使用当前的全局设置。同样，如果这是初始化模型跟踪设置的第一个请求，对于请求中未指定的跟踪设置，将使用当前的全局设置。

## GRPC

对于跟踪扩展，Triton 实现了以下 API：

```
service GRPCInferenceService
{
  …

  // 更新和获取 Triton 服务器的跟踪设置。
  rpc TraceSetting(TraceSettingRequest)
          returns (TraceSettingResponse) {}
}
```

跟踪设置 API 返回最新的跟踪设置。错误由请求返回的 google.rpc.Status 指示。OK 代码表示成功，其他代码表示失败。跟踪设置的请求和响应消息是：

```
message TraceSettingRequest
{
  // 与跟踪设置关联的值。
  // 如果未提供值，将清除设置并使用全局设置值。
  message SettingValue
  {
    repeated string value = 1;
  }

  // 要更新的新设置值，
  // 未指定的设置将保持不变。
  map<string, SettingValue> settings = 1;

  // 要应用新跟踪设置的模型名称。
  // 如果未提供，新设置将全局应用。
  string model_name = 2;
}

message TraceSettingResponse
{
  message SettingValue
  {
    repeated string value = 1;
  }

  // 最新的跟踪设置。
  map<string, SettingValue> settings = 1;
}
```

跟踪设置在[跟踪设置响应 JSON 对象](#跟踪设置响应-json-对象)中提到。注意，如果这是初始化模型跟踪设置的第一个请求，对于请求中未指定的跟踪设置，值将从当前的全局设置中复制。