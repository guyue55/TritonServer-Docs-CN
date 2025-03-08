<!--
# Copyright (c) 2022-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 日志扩展

本文档描述了 Triton 的日志扩展。日志扩展使客户端能够在 Triton 运行期间配置日志设置。Triton 在其服务器元数据的扩展字段中报告 "logging"。

## HTTP/REST

在本文档中显示的所有 JSON 模式中，`$number`、`$string`、`$boolean`、`$object` 和 `$array` 指的是基本 JSON 类型。#optional 表示可选的 JSON 字段。

Triton 在以下 URL 暴露日志端点。客户端可以使用 HTTP GET 请求获取当前的日志设置。HTTP POST 请求将修改日志设置，成功时端点将返回更新后的日志设置，失败时将返回错误。

```
GET v2/logging

POST v2/logging
```

### 日志设置响应 JSON 对象

成功的日志设置请求由 200 HTTP 状态码表示。响应对象（标识为 `$log_setting_response`）在每个成功的日志设置请求的 HTTP 正文中返回。

```
$log_setting_response =
{
  $log_setting, ...
}

$log_setting = $string : $string | $boolean | $number
```

每个 `$log_setting` JSON 描述一个"名称"/"值"对，其中"名称"是日志设置的 `$string` 表示，"值"是设置值的 `$string`、`$bool` 或 `$number` 表示。目前，定义了以下日志设置：

- "log_file"：一个 `$string` 类型的日志文件位置，日志输出将保存在此处。如果为空，日志输出将流向控制台。

- "log_info"：一个 `$boolean` 参数，控制 Triton 服务器是否记录 INFO 级别的消息。

- "log_warning"：一个 `$boolean` 参数，控制 Triton 服务器是否记录 WARNING 级别的消息。

- "log_error"：一个 `$boolean` 参数，控制 Triton 服务器是否记录 ERROR 级别的消息。

- "log_verbose_level"：一个 `$number` 参数，控制 Triton 服务器是否输出不同程度的详细消息。此值可以是任何 >= 0 的整数。如果 "log_verbose_level" 为 0，详细日志记录将被禁用，Triton 服务器将不会输出任何详细消息。如果 "log_verbose_level" 为 1，Triton 服务器将输出级别 1 的详细消息。如果 "log_verbose_level" 为 2，Triton 服务器将输出所有级别 <= 2 的详细消息，以此类推。尝试将 "log_verbose_level" 设置为小于 0 的数字将导致错误。

- "log_format"：一个 `$string` 参数，控制 Triton 服务器日志消息的格式。目前有 2 种格式："default" 和 "ISO8601"。

### 日志设置响应 JSON 错误对象

失败的日志设置请求将由 HTTP 错误状态（通常为 400）表示。HTTP 正文将包含一个 `$log_setting_error_response` 对象。

```
$log_setting_error_response =
{
  "error": $string
}
```