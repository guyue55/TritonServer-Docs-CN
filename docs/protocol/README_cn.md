<!--
# Copyright 2020-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# HTTP/REST 和 GRPC 协议

本目录包含与 Triton 使用的 HTTP/REST 和 GRPC 协议相关的文档。Triton 使用 [KServe 社区标准推理协议](https://github.com/kserve/kserve/tree/master/docs/predict-api/v2)，并在以下文档中定义了几个扩展：

- [二进制张量数据扩展](./extension_binary_data_cn.md)
- [分类扩展](./extension_classification_cn.md)
- [调度策略扩展](./extension_schedule_policy.md)
- [序列扩展](./extension_sequence.md)
- [共享内存扩展](./extension_shared_memory.md)
- [模型配置扩展](./extension_model_configuration.md)
- [模型仓库扩展](./extension_model_repository.md)
- [统计扩展](./extension_statistics.md)
- [追踪扩展](./extension_trace.md)
- [日志扩展](./extension_logging.md)
- [参数扩展](./extension_parameters.md)

请注意，一些扩展在推理协议中引入了新的字段，而其他扩展则定义了 Triton 遵循的新协议，详细信息请参阅扩展文档。

对于 GRPC 协议，[protobuf 规范](https://github.com/triton-inference-server/common/blob/main/protobuf/grpc_service.proto)也是可用的。此外，您可以在[这里](https://github.com/triton-inference-server/common/blob/main/protobuf/health.proto)找到 GRPC 健康检查协议的 protobuf 规范。

## 受限协议

您可以配置实现这些协议的 Triton 端点，以限制对某些协议的访问并控制网络设置，详细信息请参阅[协议定制指南](https://github.com/triton-inference-server/server/blob/main/docs/customization_guide/inference_protocols_cn.md#httprest-和-grpc-协议)。

## IPv6

假设您的主机或 [docker 配置](https://docs.docker.com/config/daemon/ipv6/)支持 IPv6 连接，可以按如下方式配置 `tritonserver` 使用 IPv6 HTTP 端点：
```
$ tritonserver ... --http-address ipv6:[::1]&
...
I0215 21:04:11.572305 571 grpc_server.cc:4868] Started GRPCInferenceService at 0.0.0.0:8001
I0215 21:04:11.572528 571 http_server.cc:3477] Started HTTPService at ipv6:[::1]:8000
I0215 21:04:11.614167 571 http_server.cc:184] Started Metrics Service at ipv6:[::1]:8002
```

可以通过 `netstat` 确认，例如：
```
$ netstat -tulpn | grep tritonserver
tcp6      0      0 :::8000      :::*      LISTEN      571/tritonserver
tcp6      0      0 :::8001      :::*      LISTEN      571/tritonserver
tcp6      0      0 :::8002      :::*      LISTEN      571/tritonserver
```

并可以通过 `curl` 进行测试，例如：
```
$ curl -6 --verbose "http://[::1]:8000/v2/health/ready"
*   Trying ::1:8000...
* TCP_NODELAY set
* Connected to ::1 (::1) port 8000 (#0)
> GET /v2/health/ready HTTP/1.1
> Host: [::1]:8000
> User-Agent: curl/7.68.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Content-Length: 0
< Content-Type: text/plain
<
```