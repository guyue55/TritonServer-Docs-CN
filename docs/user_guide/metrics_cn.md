<!--
# Copyright 2018-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 指标

Triton 提供 [Prometheus](https://prometheus.io/) 指标来显示 GPU 和请求统计信息。默认情况下，这些指标可以通过 http://localhost:8002/metrics 访问。这些指标仅通过访问端点获得，不会推送或发布到任何远程服务器。指标格式为纯文本，因此您可以直接查看，例如：

```
$ curl localhost:8002/metrics
```

可以使用 `tritonserver --allow-metrics=false` 选项禁用所有指标报告，而 `--allow-gpu-metrics=false` 和 `--allow-cpu-metrics=false` 可以分别用于禁用 GPU 和 CPU 指标。

可以使用 `--metrics-port` 选项选择不同的端口。默认情况下，当启用 http 服务时，Triton 会重用 `--http-address` 选项作为指标端点，并将 http 和指标端点绑定到相同的特定地址。如果未启用 http 服务，指标地址将默认绑定到 `0.0.0.0`。要唯一指定指标端点，可以使用 `--metrics-address` 选项。有关这些 CLI 选项的更多信息，请参阅 `tritonserver --help` 输出。

要更改轮询/更新指标的间隔，请参阅 `--metrics-interval-ms` 标志。"每个请求"更新的指标不受此间隔设置的影响。此间隔仅适用于以下各节表格中指定为"每个间隔"的指标：

- [推理请求指标](#推理请求指标)
- [GPU 指标](#gpu-指标)
- [CPU 指标](#cpu-指标)
- [固定内存指标](#固定内存指标)
- [响应缓存指标](#响应缓存指标)
- [自定义指标](#自定义指标)

## 推理请求指标

### 计数

对于不支持批处理的模型，*请求计数*、*推理计数*和*执行计数*将相等，表示每个推理请求都是单独执行的。

对于支持批处理的模型，可以通过计数指标来确定平均批处理大小，即*推理计数* / *执行计数*。以下示例说明了计数指标：

* 客户端发送单个批次大小为 1 的推理请求。*请求计数* = 1，*推理计数* = 1，*执行计数* = 1。

* 客户端发送单个批次大小为 8 的推理请求。*请求计数* = 1，*推理计数* = 8，*执行计数* = 1。

* 客户端发送 2 个请求：批次大小为 1 和 8。模型未启用动态批处理。*请求计数* = 2，*推理计数* = 9，*执行计数* = 2。

* 客户端发送 2 个请求：批次大小均为 1。模型启用了动态批处理，服务器将这 2 个请求动态批处理。*请求计数* = 2，*推理计数* = 2，*执行计数* = 1。

* 客户端发送 2 个请求：批次大小为 1 和 8。模型启用了动态批处理，服务器将这 2 个请求动态批处理。*请求计数* = 2，*推理计数* = 9，*执行计数* = 1。

|类别      |指标          |指标名称 |描述                            |粒度|频率    |
|--------------|----------------|------------|---------------------------|-----------|-------------|
|计数         |成功计数   |`nv_inference_request_success` |Triton 接收的成功推理请求数量（每个请求计为 1，即使请求包含批处理） |每个模型  |每个请求  |
|              |失败计数   |`nv_inference_request_failure` |Triton 接收的失败推理请求数量（每个请求计为 1，即使请求包含批处理） |每个模型  |每个请求  |
|              |推理计数 |`nv_inference_count` |执行的推理数量（批次大小为"n"计为"n"次推理，不包括缓存请求）|每个模型|每个请求|
|              |执行计数 |`nv_inference_exec_count` |推理批处理执行次数（参见[推理请求指标](#推理请求指标)，不包括缓存请求）|每个模型|每个请求|