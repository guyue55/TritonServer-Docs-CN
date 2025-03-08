<!--
# Copyright (c) 2020, NVIDIA CORPORATION. All rights reserved.
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

# 共享内存扩展

本文档描述了 Triton 的共享内存扩展。共享内存扩展允许客户端通过系统或 CUDA 共享内存来传输输入和输出张量。对于某些用例，使用共享内存而不是通过 GRPC 或 REST 接口发送张量数据可以提供显著的性能改进。由于支持这两种扩展，Triton 在其服务器元数据的扩展字段中报告 "system_shared_memory" 和 "cuda_shared_memory"。

共享内存扩展使用一组通用参数来指示输入或输出张量是通过共享内存传输的。这些参数及其类型是：

- "shared_memory_region"：字符串值，是先前注册的共享内存区域的名称。系统共享内存区域和 CUDA 共享内存区域的区域名称共享一个命名空间。

- "shared_memory_offset"：int64 值，是张量数据在区域中开始的字节偏移量。

- "shared_memory_byte_size"：int64 值，是数据的字节大小。

"shared_memory_offset" 参数是可选的，默认为零。其他两个参数是必需的。如果只提供其中一个，Triton 将返回错误。

注意，Windows 目前不支持共享内存。Jetson 仅支持系统共享内存。

## HTTP/REST

在本文档中显示的所有 JSON 模式中，`$number`、`$string`、`$boolean`、`$object` 和 `$array` 指的是基本 JSON 类型。#optional 表示可选的 JSON 字段。

共享内存参数可以在 `$request_input` 参数中使用，以指示相应的输入是通过共享内存传输的。这些参数可以在 `$request_output` 参数中使用，以指示请求的输出应该通过共享内存传输。

当为输入张量设置这些参数时，`$request_input` 的 "data" 字段不能设置。如果设置了 "data" 字段，Triton 将返回错误。当为请求的输出张量设置这些参数时，返回的 `$response_output` 不能定义 "data" 字段。

共享内存区域必须由客户端创建，然后在使用 "shared_memory_region" 参数引用之前向 Triton 注册。系统和 CUDA 共享内存扩展各自需要一组不同的 API 来注册共享内存区域。

### 系统共享内存

系统共享内存扩展需要状态、注册和注销 API。

Triton 暴露以下 URL 来注册和注销系统共享内存区域。

```
GET v2/systemsharedmemory[/region/${REGION_NAME}]/status

POST v2/systemsharedmemory/region/${REGION_NAME}/register

POST v2/systemsharedmemory[/region/${REGION_NAME}]/unregister
```

#### 状态

系统共享内存状态请求通过 HTTP GET 发送到状态端点。在相应的响应中，HTTP 正文包含响应 JSON。如果在 URL 中提供了 REGION_NAME，响应将包含相应区域的状态。如果在 URL 中未提供 REGION_NAME，响应将包含所有已注册区域的状态。

成功的状态请求由 200 HTTP 状态码表示。响应对象（标识为 `$system_shared_memory_status_response`）在每个成功请求的 HTTP 正文中返回。

```
$system_shared_memory_status_response =
[
  {
    "name" : $string,
    "key" : $string,
    "offset" : $number,
    "byte_size" : $number
  },
  …
]
```

- "name"：共享内存区域的名称。

- "key"：包含共享内存区域的底层内存对象的键。

- "offset"：在底层内存对象中到共享内存区域开始的字节偏移量。

- "byte_size"：共享内存区域的大小，以字节为单位。

失败的状态请求必须由 HTTP 错误状态（通常为 400）表示。HTTP 正文必须包含 `$system_shared_memory_status_error_response` 对象。

```
$system_shared_memory_status_error_response =
{
  "error": $string
}
```

- "error"：错误的描述性消息。

#### 注册

系统共享内存注册请求通过 HTTP POST 发送到注册端点。在相应的响应中，HTTP 正文包含响应 JSON。成功的注册请求由 200 HTTP 状态码表示。

请求对象（标识为 `$system_shared_memory_register_request`）必须在 HTTP 正文中提供。

```
$system_shared_memory_register_request =
{
  "key" : $string,
  "offset" : $number,
  "byte_size" : $number
}
```

- "key"：包含共享内存区域的底层内存对象的键。

- "offset"：在底层内存对象中到共享内存区域开始的字节偏移量。

- "byte_size"：共享内存区域的大小，以字节为单位。

失败的注册请求必须由 HTTP 错误状态（通常为 400）表示。HTTP 正文必须包含 `$system_shared_memory_register_error_response` 对象。

```
$system_shared_memory_register_error_response =
{
  "error": $string
}
```

- "error"：错误的描述性消息。

#### 注销

系统共享内存注销请求通过 HTTP POST 发送到注销端点。在请求中，HTTP 正文必须为空。

成功的注册请求由 200 HTTP 状态表示。如果在 URL 中提供了 REGION_NAME，则注销单个区域。如果在 URL 中未提供 REGION_NAME，则注销所有区域。

失败的注销请求必须由 HTTP 错误状态（通常为 400）表示。HTTP 正文必须包含 `$system_shared_memory_unregister_error_response` 对象。

```
$system_shared_memory_unregister_error_response =
{
  "error": $string
}
```

- "error"：错误的描述性消息。

### CUDA 共享内存

CUDA 共享内存扩展需要状态、注册和注销 API。

Triton 暴露以下 URL 来注册和注销系统共享内存区域。

```
GET v2/cudasharedmemory[/region/${REGION_NAME}]/status

POST v2/cudasharedmemory/region/${REGION_NAME}/register

POST v2/cudasharedmemory[/region/${REGION_NAME}]/unregister
```

#### 状态

CUDA 共享内存状态请求通过 HTTP GET 发送到状态端点。在相应的响应中，HTTP 正文包含响应 JSON。如果在 URL 中提供了 REGION_NAME，响应将包含相应区域的状态。如果在 URL 中未提供 REGION_NAME，响应将包含所有已注册区域的状态。

成功的状态请求由 200 HTTP 状态码表示。响应对象（标识为 `$cuda_shared_memory_status_response`）在每个成功请求的 HTTP 正文中返回。

```
$cuda_shared_memory_status_response =
[
  {
    "name" : $string,
    "device_id" : $number,
    "byte_size" : $number
  },
  …
]
```

- "name"：共享内存区域的名称。

- "device_id"：创建 cudaIPC 句柄的 GPU 设备 ID。

- "byte_size"：共享内存区域的大小，以字节为单位。

失败的状态请求必须由 HTTP 错误状态（通常为 400）表示。HTTP 正文必须包含 `$cuda_shared_memory_status_error_response` 对象。

```
$cuda_shared_memory_status_error_response =
{
  "error": $string
}
```

- "error"：错误的描述性消息。

#### 注册

CUDA 共享内存注册请求通过 HTTP POST 发送到注册端点。在相应的响应中，HTTP 正文包含响应 JSON。成功的注册请求由 200 HTTP 状态码表示。

请求对象（标识为 `$cuda_shared_memory_register_request`）必须在 HTTP 正文中提供。

```
$cuda_shared_memory_register_request =
{
  "raw_handle" : { "b64" : $string },
  "device_id" : $number,
  "byte_size" : $number
}
```

- "raw_handle"：序列化的 cudaIPC 句柄，base64 编码。

- "device_id"：创建 cudaIPC 句柄的 GPU 设备 ID。

- "byte_size"：共享内存区域的大小，以字节为单位。

失败的注册请求必须由 HTTP 错误状态（通常为 400）表示。HTTP 正文必须包含 `$cuda_shared_memory_register_error_response` 对象。

```
$cuda_shared_memory_register_error_response =
{
  "error": $string
}
```

- "error"：错误的描述性消息。

#### 注销

CUDA 共享内存注销请求通过 HTTP POST 发送到注销端点。在请求中，HTTP 正文必须为空。

成功的注册请求由 200 HTTP 状态表示。如果在 URL 中提供了 REGION_NAME，则注销单个区域。如果在 URL 中未提供 REGION_NAME，则注销所有区域。

失败的注