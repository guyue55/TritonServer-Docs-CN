<!--
# Copyright 2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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
### Triton 服务器 (tritonfrontend) 绑定（测试版）

`tritonfrontend` Python 包是 Triton 现有 C++ 实现的前端的一组绑定。目前，`tritonfrontend` 支持启动 `KServeHttp` 和 `KServeGrpc` 前端。这些绑定与 Triton 的 Python 进程内 API（[`tritonserver`](https://github.com/triton-inference-server/core/tree/main/python/tritonserver)）和 [`tritonclient`](https://github.com/triton-inference-server/client/tree/main/src/python/library) 结合使用，通过几行 Python 代码就可以扩展使用 Triton 的全部功能集。

让我们通过一个简单的示例来了解：
1. 首先，我们需要使用 `tritonserver` 加载所需的模型并启动服务器。
```python
import tritonserver

# 构建模型仓库路径
model_path = f"server/src/python/examples/example_model_repository"

server_options = tritonserver.Options(
    server_id="ExampleServer",
    model_repository=model_path,
    log_error=True,
    log_warn=True,
    log_info=True,
)
server = tritonserver.Server(server_options).start(wait_until_ready=True)
```
注意：根据您的设置，可能需要编辑 `model_path`。

2. 现在，使用 `tritonfrontend` 启动相应的服务
```python
from tritonfrontend import KServeHttp, KServeGrpc
http_options = KServeHttp.Options(thread_count=5)
http_service = KServeHttp.Server(server, http_options)
http_service.start()

# 默认选项（如果未提供）
grpc_service = KServeGrpc.Server(server)
grpc_service.start()
```

3. 最后，在服务运行时，我们可以使用 `tritonclient` 或简单的 `curl` 命令向前端发送请求并接收响应。

```python
import tritonclient.http as httpclient
import numpy as np # 使用版本 numpy < 2
model_name = "identity" # 输出 == 输入
url = "localhost:8000"

# 创建 Triton 客户端
client = httpclient.InferenceServerClient(url=url)

# 准备输入数据
input_data = np.array([["Roger Roger"]], dtype=object)

# 创建输入和输出对象
inputs = [httpclient.InferInput("INPUT0", input_data.shape, "BYTES")]

# 设置输入张量的数据
inputs[0].set_data_from_numpy(input_data)

results = client.infer(model_name, inputs=inputs)

# 获取输出数据
output_data = results.as_numpy("OUTPUT0")

# 打印结果
print("[推理结果]")
print("输出数据:", output_data)

# 停止相应的服务和服务器
http_service.stop()
```