<!--
# Copyright 2019-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# Triton 服务器追踪

Triton 包含为单个推理请求生成详细追踪的功能。追踪功能通过运行 tritonserver 可执行文件时的命令行参数启用。

Triton 中的 `--trace-config` 命令行选项可用于指定全局和追踪模式特定的配置设置。此标志的格式为 `--trace-config <mode>,<setting>=<value>`，其中 `<mode>` 可以是 `triton` 或 `opentelemetry`。默认情况下，追踪模式设置为 `triton`，服务器将使用 Triton 的追踪 API。对于 `opentelemetry` 模式，服务器将使用 [OpenTelemetry 的 API](#opentelemetry-trace-support) 来生成、收集和导出单个推理请求的追踪。

要指定全局追踪设置（级别、速率、计数或模式），格式为 `--trace-config <setting>=<value>`。

使用 Triton 追踪 API 的示例用法：

```
$ tritonserver \
    --trace-config triton,file=/tmp/trace.json \
    --trace-config triton,log-frequency=50 \
    --trace-config rate=100 \
    --trace-config level=TIMESTAMPS \
    --trace-config count=100 ...
```

## 追踪设置
### 全局设置
下表显示了可传递给 `--trace-config` 的可用全局追踪设置
<table>
  <thead>
  <tr>
    <th>设置</th>
    <th>默认值</th>
    <th>描述</th>
  </tr>
  </thead>
  <tbody>
    <tr>
    <td><code>rate</code></td>
    <td>1000</td>
    <td>
      指定采样率。与已弃用的 <code>--trace-rate</code> 相同。<br/>
      例如，值为 1000 表示每 1000 个推理请求将被追踪一次。
    </td>
    </tr>
    <tr>
    <td><code>level</code></td>
    <td>OFF</td>
    <td>
      指示应收集的追踪详细程度，可以多次指定以追踪多个信息。<br/>
      与已弃用的 <code>--trace-level</code> 相同。<br/>
      选项为 <code>TIMESTAMPS</code> 和 <code>TENSORS</code>。<br/>
      <b>注意</b>，<code>opentelemetry</code> 模式目前不支持 <code>TENSORS</code> 级别。
    </td>
    </tr>
    <tr>
    <td><code>count</code></td>
    <td>-1</td>
    <td>
      指定要收集的剩余追踪数量。<br/>
      默认值 -1 表示永不停止收集追踪。<br/>
      值为 100 时，Triton 将在收集 100 个追踪后停止追踪请求。<br/>
      与已弃用的 <code>--trace-count</code> 相同。
    </td>
    </tr>
    <tr>
    <td><code>mode</code></td>
    <td>triton</td>
    <td>
      指定用于收集追踪的 API。<br/>
      选项为 <code>triton</code> 或 <code>opentelemetry</code>。<br/>
    </td>
    </tr>
  </tbody>
</table>

### Triton 追踪 API 设置

下表显示了 `--trace-config triton,<setting>=<value>` 的可用 Triton 追踪 API 设置。
<table>
  <thead>
  <tr>
    <th>设置</th>
    <th>默认值</th>
    <th>描述</th>
  </tr>
  </thead>
  <tbody>
    <tr>
    <td><code>file</code></td>
    <td>空字符串</td>
    <td>
      指示追踪输出应写入的位置。<br/>
      与已弃用的 <code>--trace-file</code> 相同。<br/>
    </td>
    </tr>
    <tr>
    <td><code>log-frequency</code></td>
    <td>0</td>
    <td>
      指定追踪写入文件的频率。<br/>
      例如，值为 50 表示 Triton 将每收集 50 个追踪就记录到文件中。<br/>
      与已弃用的 <code>--trace-log-frequency</code> 相同。<br/>
    </td>
    </tr>
  </tbody>
</table>

除了命令行中的追踪配置设置外，您还可以使用[追踪协议](../protocol/extension_trace.md)修改追踪配置。当追踪模式设置为 `opentelemetry` 时，此选项当前不受支持。

**注意**：以下标志已**弃用**：

`--trace-file` 选项指示追踪输出应写入的位置。`--trace-rate` 选项指定采样率。在此示例中，每 100 个推理请求将被追踪一次。`--trace-level` 选项指示应收集的追踪详细程度。`--trace-level` 选项可以多次指定以追踪多个信息。`--trace-log-frequency` 选项指定追踪写入文件的频率。在此示例中，Triton 将每收集 50 个追踪就记录到文件中。`--trace-count` 选项指定要收集的剩余追踪数量。在此示例中，Triton 将在收集 100 个追踪后停止追踪更多请求。使用 `--help` 选项获取更多信息。

## 支持的追踪级别选项

- `TIMESTAMPS`：追踪每个请求的执行时间戳。
- `TENSORS`：追踪执行过程中的输入和输出张量。

## JSON 追踪输出

追踪输出是具有以下架构的 JSON 文件。

```
[
  {
    "model_name": $string,
    "model_version": $number,
    "id": $number,
    "request_id": $string,
    "parent_id": $number
  },
  {
    "id": $number,
    "timestamps": [
      { "name" : $string, "ns" : $number }
    ]
  },
  {
    "id": $number
    "activity": $string,
    "tensor":{
      "name": $string,
      "data": $string,
      "shape": $string,
      "dtype": $string
    }
  },
  ...
]
```

每个追踪都被分配一个 "id"，表示推理请求的模型名称和版本。如果追踪来自作为集成的一部分运行的模型，"parent_id" 将指示包含集成的 "id"。
例如：
```
[
  {
    "id": 1,
    "model_name": "simple",
    "model_version": 1
  },
  ...
]
```

每个 `TIMESTAMPS` 追踪将有一个或多个 "timestamps"，每个时间戳都有一个名称和以纳秒为单位的时间戳（"ns"）。
例如：

```
[
  {"id": 1, "timestamps": [{ "name": "HTTP_RECV_START", "ns": 2356425054587444 }] },
  {"id": 1, "timestamps": [{ "name": "HTTP_RECV_END", "ns": 2356425054632308 }] },
  {"id": 1, "timestamps": [{ "name": "REQUEST_START", "ns": 2356425054785863 }] },
  {"id": 1, "timestamps": [{ "name": "QUEUE_START", "ns": 2356425054791517 }] },
  {"id": 1, "timestamps": [{ "name": "INFER_RESPONSE_COMPLETE", "ns": 2356425057587919 }] },
  {"id": 1, "timestamps": [{ "name": "COMPUTE_START", "ns": 2356425054887198 }] },
  {"id": 1, "timestamps": [{ "name": "COMPUTE_INPUT_END", "ns": 2356425057152908 }] },
  {"id": 1, "timestamps": [{ "name": "COMPUTE_OUTPUT_START", "ns": 2356425057497763 }] },
  {"id": 1, "timestamps": [{ "name": "COMPUTE_END", "ns": 2356425057540989 }] },
  {"id": 1, "timestamps": [{ "name": "REQUEST_END", "ns": 2356425057643164 }] },
  {"id": 1, "timestamps": [{ "name": "HTTP_SEND_START", "ns": 2356425057681578 }] },
  {"id": 1, "timestamps": [{ "name": "HTTP_SEND_END", "ns": 2356425057712991 }] }
]
```

每个 `TENSORS` 追踪将包含一个 "activity" 和一个 "tensor"。"activity" 指示张量的类型，目前包括 "TENSOR_QUEUE_INPUT" 和 "TENSOR_BACKEND_OUTPUT"。"tensor" 包含张量的详细信息，包括其 "name"、"data" 和 "dtype"。例如：

```
[
  {
    "id": 1,
    "activity": "TENSOR_QUEUE_INPUT",
    "tensor":{
      "name": "input",
      "data": "0.1,0.1,0.1,...",
      "shape": "1,16",
      "dtype": "FP32"
    }
  }
]
```

## 追踪摘要工具

可以使用示例[追踪摘要工具](https://github.com/triton-inference-server/server/blob/main/qa/common/trace_summary.py)来汇总从 Triton 收集的一组追踪。基本用法是：

```
$ trace_summary.py <trace file>
```

这将为文件中的所有追踪生成摘要报告。HTTP 和 GRPC 推理请求分别报告。

```
File: trace.json
Summary for simple (-1): trace count = 1
HTTP infer request (avg): 403.578us
	Receive (avg): 20.555us
	Send (avg): 4.52us
	Overhead (avg): 24.592us
	Handler (avg): 353.911us
  		Overhead (avg): 23.675us
  		Queue (avg): 18.019us
  		Compute (avg): 312.217us
  			Input (avg): 24.151us
  			Infer (avg): 244.186us
  			Output (avg): 43.88us
Summary for simple (-1): trace count = 1
GRPC infer request (avg): 383.601us
	Send (avg): 62.816us
	Handler (avg): 392.924us
  		Overhead (avg): 51.968us
  		Queue (avg): 21.45us
  		Compute (avg): 319.506us
  			Input (avg): 27.76us
  			Infer (avg): 227.844us
  			Output (avg): 63.902us
```

注意：gRPC 摘要中不包含 "Receive (avg)" 指标，因为 gRPC 库不提供任何非侵入式钩子来检测从网络读取消息所花费的时间。追踪 HTTP 请求将提供从网络读