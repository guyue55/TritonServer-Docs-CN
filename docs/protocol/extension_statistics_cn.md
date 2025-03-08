<!--
# Copyright 2020-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 统计扩展

本文档描述了 Triton 的统计扩展。统计扩展支持报告每个模型（每个版本）的统计信息，这些信息提供了自 Triton 启动以来特定模型（版本）的所有活动的汇总信息。由于支持此扩展，Triton 在其服务器元数据的扩展字段中报告 "statistics"。

## HTTP/REST

在本文档中显示的所有 JSON 模式中，`$number`、`$string`、`$boolean`、`$object` 和 `$array` 指的是基本 JSON 类型。#optional 表示可选的 JSON 字段。

Triton 在以下 URL 暴露统计端点。URL 中的特定模型名称部分是可选的；如果未提供，Triton 将返回所有模型所有版本的统计信息。如果在 URL 中给出了特定模型，则版本部分是可选的；如果未提供，Triton 将返回指定模型的所有版本的统计信息。

```
GET v2/models[/${MODEL_NAME}[/versions/${MODEL_VERSION}]]/stats
```

### 统计响应 JSON 对象

成功的统计请求由 200 HTTP 状态码表示。响应对象（标识为 `$stats_model_response`）在每个成功请求的 HTTP 正文中返回。

```
$stats_model_response =
{
  "model_stats" : [ $model_stat, ... ]
}
```

每个 `$model_stat` 对象给出特定模型和版本的统计信息。对于不支持版本的服务器，`$version` 字段是可选的。

```
$model_stat =
{
  "name" : $string,
  "version" : $string #optional,
  "last_inference" : $number,
  "inference_count" : $number,
  "execution_count" : $number,
  "inference_stats" : $inference_stats,
  "response_stats" : { $string : $response_stats, ... },
  "batch_stats" : [ $batch_stats, ... ],
  "memory_usage" : [ $memory_usage, ...]
}
```

- "name"：模型的名称。

- "version"：模型的版本。

- "last_inference"：为此模型发出的最后一个推理请求的时间戳，以毫秒为单位（从纪元开始）。

- "inference_count"：为此模型发出的成功推理请求的累计计数。批处理请求中的每个推理都被计为单独的推理。例如，如果客户端发送一个批量大小为 64 的单个推理请求，"inference_count" 将增加 64。同样，如果客户端发送 64 个批量大小为 1 的单独请求，"inference_count" 也将增加 64。"inference_count" 值不包括缓存命中。

- "execution_count"：为模型执行的成功推理执行的累计计数。当启用动态批处理时，单个模型执行可以为多个推理请求执行推理。例如，如果客户端发送 64 个批量大小为 1 的单独请求，并且动态批处理器将它们批处理成一个大批量进行模型执行，则 "execution_count" 将增加 1。相反，如果未为该模型启用动态批处理，则每个 64 个单独请求都独立执行，此时 "execution_count" 将增加 64。"execution_count" 值不包括缓存命中。

- "inference_stats"：模型的聚合统计信息。例如，"inference_stats":"success" 表示模型的成功推理请求数。

- "response_stats"：模型的聚合响应统计信息。例如，{ "key" : { "response_stats" : "success" } } 表示模型在 "key" 处的成功响应的聚合统计信息，其中 "key" 标识模型在不同请求中生成的每个响应。例如，对于生成三个响应的模型，键可以是 "0"、"1" 和 "2"，按顺序标识这三个响应。

- "batch_stats"：模型中执行的每个不同批量大小的聚合统计信息。批处理统计信息指示实际执行了多少模型执行，并显示由于不同批量大小导致的差异（例如，较大的批量通常需要更长时间来计算）。

- "memory_usage"：在模型加载期间检测到的内存使用情况，可用于估计模型卸载后将释放的内存。请注意，估计是由分析工具和框架的内存模式推断的，因此建议进行实验以了解可以依赖报告的内存使用情况的场景。作为起点，ONNX Runtime 后端和 TensorRT 后端中模型的 GPU 内存使用通常是一致的。

```
$inference_stats =
{
  "success" : $duration_stat,
  "fail" : $duration_stat,
  "queue" : $duration_stat,
  "compute_input" : $duration_stat,
  "compute_infer" : $duration_stat,
  "compute_output" : $duration_stat,
  "cache_hit": $duration_stat,
  "cache_miss": $duration_stat
}
```

- "success"：所有成功推理请求的计数和累计持续时间。"success" 计数和累计持续时间包括缓存命中。

- "fail"：所有失败推理请求的计数和累计持续时间。

- "queue"：推理请求在调度或其他队列中等待的计数和累计持续时间。"queue" 计数和累计持续时间包括缓存命中。

- "compute_input"：准备模型框架/后端所需的输入张量数据的计数和累计持续时间。例如，此持续时间应包括将输入张量数据复制到 GPU 的时间。"compute_input" 计数和累计持续时间不包括缓存命中。

- "compute_infer"：执行模型的计数和累计持续时间。"compute_infer" 计数和累计持续时间不包括缓存命中。

- "compute_output"：提取模型框架/后端产生的输出张量数据的计数和累计持续时间。例如，此持续时间应包括将输出张量数据从 GPU 复制的时间。"compute_output" 计数和累计持续时间不包括缓存命中。

- "cache_hit"：响应缓存命中的计数和在缓存命中时查找和提取输出张量数据的累计持续时间。例如，此持续时间应包括将输出张量数据从响应缓存复制到响应对象的时间。

- "cache_miss"：响应缓存未命中的计数和在缓存未命中时查找和插入输出张量数据到响应缓存的累计持续时间。例如，此持续时间应包括将输出张量数据从响应对象复制到响应缓存的时间。

```
$response_stats =
{
  "compute_infer" : $duration_stat,
  "compute_output" : $duration_stat,
  "success" : $duration_stat,
  "fail" : $duration_stat,
  "empty_response" : $duration_stat,
  "cancel" : $duration_stat
}
```

- "compute_infer"：计算响应的计数和累计持续时间。
- "compute_output"：提取计算响应的输出张量的计数和累计持续时间。
- "success"：成功推理的计数和累计持续时间。持续时间是推理和输出持续时间的总和。
- "fail"：失败推理的计数和累计持续时间。持续时间是推理和输出持续时间的总和。
- "empty_response"：空响应/无响应推理的计数和累计持续时间。持续时间是推理持续时间。
- "cancel"：推理取消的计数和累计持续时间。持续时间用于清理被取消的推理请求所持有的资源。

```
$batch_stats =
{
  "batch_size" : $number,
  "compute_input" : $duration_stat,
  "compute_infer" : $duration_stat,
  "compute_output" : $duration_stat
}
```

- "batch_size"：批量的大小。

- "count"：在模型上执行批量大小的次数。单个模型执行为整个请求批量执行推理，如果启用了动态批处理，还可以为多个请求执行推理。

- "compute_input"：使用给定批量大小准备模型框架/后端所需的输入张量数据的计数和累计持续时间。例如，此持续时间应包括将输入张量数据复制到 GPU 的时间。

- "compute_infer"：使用给定批量大小执行模型的计数和累计持续时间。

- "compute_output"：使用给定批量大小提取模型框架/后端产生的输出张量数据的计数和累计持续时间。例如，此持续时间应包括将输出张量数据从 GPU 复制的时间。

`$duration_stat` 对象报告计数和总时间。这种格式可以被采样以确定不仅是长期运行的平均值，还包括采样点之间的增量平均值。

```
$duration_stat =
{
  "count" : $number,
  "ns" : $number
}
```

- "count"：收集统计信息的次数。

- "ns"：统计信息的总持续时间，以纳秒为单位。

```
$memory_usage =
{
  "type" : $string,
  "id" : $number,
  "byte_size" : $number
}
```

- "type"：内存类型，值可以是 "CPU"、"CPU_PINNED"、"GPU"。

- "id"：内存的 ID，通常与 "type" 一起使用来标识托管内存的设备。

- "byte_size"：内存的字节大小。

### 统计响应 JSON 错误对象

失败的统计请求将由 HTTP 错误状态（通常为 400）表示。HTTP 正文必须包含 `$repository_statistics_error_response` 对象。

```
$repository_statistics_error_response =
{
  "error": $string