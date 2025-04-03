<!--
# Copyright (c) 2021-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 在 PAI-EAS 上部署 Triton 推理服务器
* 目录
   - [描述](https://yuque.alibaba-inc.com/pai/blade/mtptqc#Description)
   - [前提条件](https://yuque.alibaba-inc.com/pai/blade/mtptqc#Prerequisites)
   - [演示说明](https://yuque.alibaba-inc.com/pai/blade/mtptqc#31bb94ef)
   - [其他资源](https://yuque.alibaba-inc.com/pai/blade/mtptqc#89d5e680)
   - [已知问题](https://yuque.alibaba-inc.com/pai/blade/mtptqc#558ab0be)

# 描述
本仓库包含了如何在阿里云的 EAS（弹性算法服务）中部署 NVIDIA Triton 推理服务器的信息。
- EAS 为深度学习开发者提供了在阿里云上部署模型的简单方法。
- 在 EAS 上使用 **Triton Processor** 是部署 Triton 推理服务器的推荐方式。用户只需准备好模型，并通过将处理器类型设置为 `triton` 来创建 EAS 服务即可。
- 模型需要上传到阿里云的 OSS（对象存储服务）。用户在 OSS 中的模型仓库将被挂载到 Triton 服务器可见的本地路径。
- 本文档使用 Triton 自带的示例模型进行演示。可以通过 `fetch_models.sh` 脚本下载 tensorflow inception 模型。

# 前提条件
- 您需要注册一个阿里云账号，并能够使用 [eascmd](https://help.aliyun.com/document_detail/111031.html?spm=a2c4g.11186623.6.752.42356f46FN5fU1)，这是一个用于在 EAS 上创建、停止或扩展服务的命令行工具。
- 在创建 EAS 服务之前，您需要按照此[文档](https://www.alibabacloud.com/help/doc-detail/120122.htm)在 EAS 上购买专用资源组（CPU 或 GPU）。
- 确保您可以使用 OSS（对象存储服务），模型需要上传到您自己的 OSS 存储桶中。

# 演示说明
## 在 OSS 中准备模型仓库目录
通过 [fetch_model.sh](https://github.com/triton-inference-server/server/blob/main/docs/examples/fetch_models.sh) 下载 tensorflow inception 模型。然后使用 [ossutil](https://help.aliyun.com/document_detail/50452.html?spm=a2c4g.11186623.6.833.26d66d51dPEytI)（这是一个用于使用 OSS 的命令行工具）将模型上传到您想要的 OSS 目录中。

```
./ossutil cp inception_graphdef/ oss://triton-model-repo/models
```
## 使用 eascmd 通过 json 配置创建 Triton 服务
以下是我们在 EAS 上创建 Triton 服务器时使用的 json 配置。
```
{
  "name": "<your triton service name>",
  "processor": "triton",
  "processor_params": [
    "--model-repository=oss://triton-model-repo/models",
    "--allow-grpc=true",
    "--allow-http=true"
  ],
  "metadata": {
    "instance": 1,
    "cpu": 4,
    "gpu": 1,
    "memory": 10000,
    "resource": "<your resource id>",
    "rpc.keepalive": 3000
  }
}
```
只有 processor 和 processor_params 与普通的 EAS 服务不同。
|参数|详细信息|
|--------|-------|
|processor|名称应为 **triton** 以在 EAS 上使用 Triton|
|processor_params|字符串列表，每个元素都是 tritonserver 的一个参数|

```
./eascmd create triton.config
[RequestId]: AECDB6A4-CB69-4688-AA35-BA1E020C39E6
+-------------------+------------------------------------------------------------------------------------------------+
| Internet Endpoint | http://1271520832287160.cn-shanghai.pai-eas.aliyuncs.com/api/predict/test_triton_processor     |
| Intranet Endpoint | http://1271520832287160.vpc.cn-shanghai.pai-eas.aliyuncs.com/api/predict/test_triton_processor |
|             Token | MmY3M2ExZGYwYjZiMTQ5YTRmZWE3MDAzNWM1ZTBiOWQ3MGYxZGNkZQ==                                       |
+-------------------+------------------------------------------------------------------------------------------------+
[OK] Service is now deploying
[OK] Successfully synchronized resources
[OK] Waiting [Total: 1, Pending: 1, Running: 0]
[OK] Waiting [Total: 1, Pending: 1, Running: 0]
[OK] Running [Total: 1, Pending: 0, Running: 1]
[OK] Service is running
```
## 使用 Python 客户端查询 Triton 服务
### 安装 Triton 的 Python 客户端
```
pip install tritonclient[all]
```
### 查询 inception 模型的示例
```
import numpy as np
import time
from PIL import Image

import tritonclient.http as httpclient
from tritonclient.utils import InferenceServerException

URL = "<servcice url>"
HEADERS = {"Authorization": "<service token>"}
input_img = httpclient.InferInput("input", [1, 299, 299, 3], "FP32")
# 使用 imagenet 中的一张猫的图片或者您喜欢的任何猫的图片
img = Image.open('./cat.png').resize((299, 299))
img = np.asarray(img).astype('float32') / 255.0
input_img.set_data_from_numpy(img.reshape([1, 299, 299, 3]), binary_data=True)

output = httpclient.InferRequestedOutput(
    "InceptionV3/Predictions/Softmax", binary_data=True
)
triton_client = httpclient.InferenceServerClient(url=URL, verbose=False)

start = time.time()
for i in range(10):
    results = triton_client.infer(
        "inception_graphdef", inputs=[input_img], outputs=[output], headers=HEADERS
    )
    res_body = results.get_response()
    elapsed_ms = (time.time() - start) * 1000
    if i == 0:
        print("model name: ", res_body["model_name"])
        print("model version: ", res_body["model_version"])
        print("output name: ", res_body["outputs"][0]["name"])
        print("output shape: ", res_body["outputs"][0]["shape"])
    print("[{}] Avg rt(ms): {:.2f}".format(i, elapsed_ms))
    start = time.time()
```
运行 Python 脚本后，您将得到以下结果：
```
[0] Avg rt(ms): 86.05
[1] Avg rt(ms): 52.35
[2] Avg rt(ms): 50.56
[3] Avg rt(ms): 43.45
[4] Avg rt(ms): 41.19
[5] Avg rt(ms): 40.55
[6] Avg rt(ms): 37.24
[7] Avg rt(ms): 37.16
[8] Avg rt(ms): 36.68
[9] Avg rt(ms): 34.24
[10] Avg rt(ms): 34.27
```
# 其他资源
查看以下资源以了解更多关于如何使用阿里云的 OSS 或 EAS 的信息。
- [阿里云 OSS 文档](https://help.aliyun.com/product/31815.html?spm=a2c4g.11186623.6.540.3c0f62e7q3jw8b)


# 已知问题
- [二进制张量数据扩展](https://github.com/triton-inference-server/server/blob/main/docs/protocol/extension_binary_data.md)尚未完全支持，需要使用支持二进制扩展的服务的用户，目前仅在 PAI-EAS 的 cn-shanghai 区域可用。
- 目前仅支持 HTTP/1，因此在查询 EAS 上的 Triton 服务器时无法使用 gRPC。HTTP/2 将在短期内得到官方支持。
- 用户在启动 Triton processor 时不应挂载整个 OSS 存储桶，而应该挂载存储桶中任意深度的子目录。否则挂载路径将不符合预期。
- 并非所有 Triton 服务器参数都在 EAS 上支持，以下参数在 EAS 上受支持：
```
model-repository
log-verbose
log-info
log-warning
log-error
exit-on-error
strict-model-config
strict-readiness
allow-http
http-thread-count
pinned-memory-pool-byte-size
cuda-memory-pool-byte-size
min-supported-compute-capability
buffer-manager-thread-count
backend-config
```