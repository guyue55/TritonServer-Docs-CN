<!--
# Copyright 2021, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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
# MLflow Triton

MLflow 插件，用于将您的模型从 MLflow 部署到 Triton 推理服务器。包含用于发布模型的脚本，这些模型采用 Triton 可识别的结构，可发布到您的 MLflow 模型注册表。

### 支持的类型

MLFlow Triton 插件目前支持以下类型，您可以根据要部署的模型在下面的示例中替换类型规范。

* onnx
* triton

## 要求

* MLflow
* Triton Python HTTP 客户端
* Triton 推理服务器

## 安装

可以使用以下命令从源代码安装插件

```
python setup.py install
```

## 快速入门

在本文档中，我们将使用 `examples` 中的文件来展示插件如何与 Triton 推理服务器交互。`examples` 中的 `onnx_float32_int32_int32` 模型是一个简单的模型，它接受两个形状为 [-1, 16] 的 float32 输入（INPUT0 和 INPUT1），并产生两个 int32 输出（OUTPUT0 和 OUTPUT1），其中 OUTPUT0 是 INPUT0 和 INPUT1 的逐元素求和，OUTPUT1 是 INPUT0 和 INPUT1 的逐元素相减。

### 在 EXPLICIT 模式下启动 Triton 推理服务器

MLflow Triton 插件必须与正在运行的 Triton 服务器一起工作，有关如何启动服务器的信息，请参见 Triton 推理服务器的[文档](https://github.com/triton-inference-server/server/blob/main/docs/getting_started/quickstart.md)。注意，服务器应该在 EXPLICIT 模式下运行（`--model-control-mode=explicit`），以利用插件的部署功能。

服务器启动后，必须设置以下环境变量，以便插件能够正确与服务器交互：
* `TRITON_URL`：Triton HTTP 端点的地址
* `TRITON_MODEL_REPO`：Triton 模型仓库的路径。它可以是 s3 URI，但请记住需要设置环境变量 AWS_ACCESS_KEY_ID 和 AWS_SECRET_ACCESS_KEY。

### 将模型发布到 MLflow

#### ONNX 类型

MLFlow ONNX 内置功能可用于直接将 `onnx` 类型模型发布到 MLFlow，MLFlow Triton 插件将准备 Triton 期望格式的模型。您还可以将 [`config.pbtxt`](https://github.com/triton-inference-server/server/blob/main/docs/protocol/extension_model_configuration.md) 作为额外的工件记录，Triton 将用它来提供模型服务。否则，服务器应该在启用自动完成功能的情况下运行（`--strict-model-config=false`）以生成模型配置。

```
import mlflow.onnx
import onnx
model = onnx.load("examples/onnx_float32_int32_int32/1/model.onnx")
mlflow.onnx.log_model(model, "triton", registered_model_name="onnx_float32_int32_int32")
```

#### Triton 类型

对于 Triton 支持但 MLFlow Triton 插件尚未识别的其他模型框架，可以使用 `publish_model_to_mlflow.py` 脚本将 `triton` 类型模型发布到 MLflow。`triton` 类型模型是一个包含按照[模型布局](https://github.com/triton-inference-server/server/blob/main/docs/user_guide/model_repository.md#repository-layout)组织的模型文件的目录。以下是示例用法：

```
cd /scripts

python publish_model_to_mlflow.py --model_name onnx_float32_int32_int32 --model_directory <path-to-the-examples-directory>/onnx_float32_int32_int32 --flavor triton
```

### 将 MLflow 中跟踪的模型部署到 Triton

一旦模型在 MLflow 中发布和跟踪，就可以通过 MLflow 的部署命令将其部署到 Triton，以下命令将把模型下载到 Triton 的模型仓库并请求 Triton 加载模型。

```
mlflow deployments create -t triton --flavor triton --name onnx_float32_int32_int32 -m models:/onnx_float32_int32_int32/1
```

### 执行推理

模型部署后，以下命令是向部署发送推理请求的 CLI 用法。

```
mlflow deployments predict -t triton --name onnx_float32_int32_int32 --input-path <path-to-the-examples-directory>/input.json --output-path output.json
```

推理结果将写入 `output.json`，您可以将其与 `expected_output.json` 中的结果进行比较

## MLflow 部署

"MLflow 部署"是一组用于将 MLflow 模型部署到自定义服务工具的 MLflow API。MLflow Triton 插件实现了以下部署功能，以支持在 MLflow 中与 Triton 服务器的交互。

### 创建部署

MLflow deployments create API 将模型部署到 Triton 目标，这将把模型下载到 Triton 的模型仓库并请求 Triton 加载模型。

使用 CLI 创建 MLflow 部署：

```
mlflow deployments create -t triton --flavor triton --name model_name -m models:/model_name/1
```

使用 Python API 创建 MLflow 部署：

```
from mlflow.deployments import get_deploy_client
client = get_deploy_client('triton')
client.create_deployment("model_name", "models:/model_name/1", flavor="triton")
```

### 删除部署

MLflow deployments delete API 从 Triton 目标中删除现有部署，这将删除 Triton 模型仓库中的模型并请求 Triton 卸载模型。

使用 CLI 删除 MLflow 部署

```
mlflow deployments delete -t triton --name model_name
```

使用 Python API 删除 MLflow 部署

```
from mlflow.deployments import get_deploy_client
client = get_deploy_client('triton')
client.delete_deployment("model_name")
```

### 更新部署

MLflow deployments update API 使用 MLflow 中跟踪的另一个模型（版本）更新现有部署，这将覆盖 Triton 模型仓库中的模型并请求 Triton 重新加载模型。

使用 CLI 更新 MLflow 部署

```
mlflow deployments update -t triton --flavor triton --name model_name -m models:/model_name/2
```

使用 Python API 更新 MLflow 部署

```
from mlflow.deployments import get_deploy_client
client = get_deploy_client('triton')
client.update_deployment("model_name", "models:/model_name/2", flavor="triton")
```

### 列出部署

MLflow deployments list API 列出 Triton 目标中的所有现有部署。

使用 CLI 列出所有 MLflow 部署

```
mlflow deployments list -t triton
```

使用 Python API 列出所有 MLflow 部署

```
from mlflow.deployments import get_deploy_client
client = get_deploy_client('triton')
client.list_deployments()
```

### 获取部署

MLflow deployments get API 返回有关 Triton 目标中特定部署的信息。

使用 CLI 列出特定 MLflow 部署
```
mlflow deployments get -t triton --name model_name
```

使用 Python API 列出特定 MLflow 部署
```
from mlflow.deployments import get_deploy_client
client = get_deploy_client('triton')
client.get_deployment("model_name")
```

### 在部署上运行推理

MLflow deployments predict API 通过准备并发送请求到 Triton 来运行推理，并返回 Triton 响应。

使用 CLI 运行推理

```
mlflow deployments predict -t triton --name model_name --input-path input_file --output-path output_file
```

使用 Python API 运行推理

```
from mlflow.deployments import get_deploy_client
client = get_deploy_client('triton')
client.predict("model_name", inputs)
```