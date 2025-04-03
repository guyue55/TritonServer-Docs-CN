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

# Triton 推理服务器

[![License](https://img.shields.io/badge/License-BSD3-lightgrey.svg)](https://opensource.org/licenses/BSD-3-Clause)

> [!警告]
> 当前发布版本为 [2.51.0](https://github.com/triton-inference-server/server/releases/latest)，对应 NVIDIA GPU Cloud (NGC) 上的 24.10 容器发布版本。

Triton 推理服务器是一个开源的推理服务软件，可以简化 AI 推理过程。Triton 使团队能够部署来自多个深度学习和机器学习框架的任何 AI 模型，包括 TensorRT、TensorFlow、PyTorch、ONNX、OpenVINO、Python、RAPIDS FIL 等。Triton 推理服务器支持在 NVIDIA GPU、x86 和 ARM CPU 或 AWS Inferentia 上跨云、数据中心、边缘和嵌入式设备进行推理。Triton 推理服务器为多种查询类型提供优化的性能，包括实时、批处理、集成和音视频流。Triton 推理服务器是 [NVIDIA AI Enterprise](https://www.nvidia.com/en-us/data-center/products/ai-enterprise/) 的一部分，这是一个加速数据科学流程并简化生产 AI 开发和部署的软件平台。

主要特性包括：

- [支持多个深度学习框架](https://github.com/triton-inference-server/backend#where-can-i-find-all-the-backends-that-are-available-for-triton)
- [支持多个机器学习框架](https://github.com/triton-inference-server/fil_backend)
- [并发模型执行](docs/user_guide/architecture_cn.md#concurrent-model-execution)
- [动态批处理](docs/user_guide/model_configuration_cn.md#dynamic-batcher)
- [序列批处理](docs/user_guide/model_configuration_cn.md#sequence-batcher)和[隐式状态管理](docs/user_guide/architecture_cn.md#implicit-state-management)用于有状态模型
- 提供[后端 API](https://github.com/triton-inference-server/backend)，允许添加自定义后端和预/后处理操作
- 支持用 Python 编写自定义后端，即[基于 Python 的后端](https://github.com/triton-inference-server/backend/blob/r24.10/docs/python_based_backends.md#python-based-backends)
- 使用[集成](docs/user_guide/architecture_cn.md#ensemble-models)或[业务逻辑脚本(BLS)](https://github.com/triton-inference-server/python_backend#business-logic-scripting)的模型流水线
- 基于社区开发的 [KServe 协议](https://github.com/kserve/kserve/tree/master/docs/predict-api/v2)的 [HTTP/REST 和 GRPC 推理协议](docs/customization_guide/inference_protocols_cn.md)
- [C API](docs/customization_guide/inference_protocols_cn.md#in-process-triton-server-api)和[Java API](docs/customization_guide/inference_protocols_cn.md#java-bindings-for-in-process-triton-server-api)允许 Triton 直接链接到您的应用程序中，用于边缘和其他进程内用例
- [指标](docs/user_guide/metrics_cn.md)显示 GPU 利用率、服务器吞吐量、服务器延迟等

**刚接触 Triton 推理服务器？**使用[这些教程](https://github.com/triton-inference-server/tutorials)开始您的 Triton 之旅！

加入 [Triton 和 TensorRT 社区](https://www.nvidia.com/en-us/deep-learning-ai/triton-tensorrt-newsletter/)，及时了解最新的产品更新、错误修复、内容、最佳实践等。需要企业支持？通过 [NVIDIA AI Enterprise 软件套件](https://www.nvidia.com/en-us/data-center/products/ai-enterprise/)可获得 Triton 推理服务器的 NVIDIA 全球支持。

## 3 个简单步骤部署模型

```bash
# 步骤 1：创建示例模型仓库
git clone -b r24.10 https://github.com/triton-inference-server/server.git
cd server/docs/examples
./fetch_models.sh

# 步骤 2：从 NGC Triton 容器启动 triton
docker run --gpus=1 --rm --net=host -v ${PWD}/model_repository:/models nvcr.io/nvidia/tritonserver:24.10-py3 tritonserver --model-repository=/models

# 步骤 3：发送推理请求
# 在另一个控制台中，从 NGC Triton SDK 容器启动 image_client 示例
docker run -it --rm --net=host nvcr.io/nvidia/tritonserver:24.10-py3-sdk
/workspace/install/bin/image_client -m densenet_onnx -c 3 -s INCEPTION /workspace/images/mug.jpg

# 推理应返回以下内容
Image '/workspace/images/mug.jpg':
    15.346230 (504) = COFFEE MUG
    13.224326 (968) = CUP
    10.422965 (505) = COFFEEPOT
```

请阅读[快速入门](docs/getting_started/quickstart_cn.md)指南以获取有关此示例的更多信息。快速入门指南还包含如何在 [仅 CPU 系统](docs/getting_started/quickstart_cn.md#run-on-cpu-only-system)上启动 Triton 的示例。刚接触 Triton 并想知道从哪里开始？观看[入门视频](https://youtu.be/NQDtfSi5QF4)。

## 示例和教程

查看 [NVIDIA LaunchPad](https://www.nvidia.com/en-us/data-center/products/ai-enterprise-suite/trial/)，免费访问在 NVIDIA 基础设施上托管的一系列 Triton 推理服务器动手实验。

特定的端到端示例（如 ResNet、BERT 和 DLRM）位于 GitHub 上的 [NVIDIA 深度学习示例](https://github.com/NVIDIA/DeepLearningExamples)页面。[NVIDIA 开发者专区](https://developer.nvidia.com/nvidia-triton-inference-server)包含额外的文档、演示和示例。

## 文档

### 构建和部署

推荐使用 Docker 镜像来构建和使用 Triton 推理服务器。

- [使用 Docker 容器安装 Triton 推理服务器](docs/customization_guide/build_cn.md#building-with-docker)（*推荐*）
- [不使用 Docker 容器安装 Triton 推理服务器](docs/customization_guide/build_cn.md#building-without-docker)
- [构建自定义 Triton 推理服务器 Docker 容器](docs/customization_guide/compose_cn.md)
- [从源代码构建 Triton 推理服务器](docs/customization_guide/build_cn.md#building-on-unsupported-platforms)
- [为 Windows 10 构建 Triton 推理服务器](docs/customization_guide/build_cn.md#building-for-windows-10)
- 在 [GCP](deploy/gcp/README_cn.md)、[AWS](deploy/aws/README_cn.md) 和 [NVIDIA FleetCommand](deploy/fleetcommand/README_cn.md) 上使用 Kubernetes 和 Helm 部署 Triton 推理服务器的示例
- [安全部署注意事项](docs/customization_guide/deploy_cn.md)

### 使用 Triton

#### 为 Triton 推理服务器准备模型

使用 Triton 服务模型的第一步是将一个或多个模型放入[模型仓库](docs/user_guide/model_repository_cn.md)中。根据模型的类型和您想为模型启用的 Triton 功能，您可能需要为模型创建[模型配置](docs/user_guide/model_configuration_cn.md)。

- [如果模型需要，添加自定义操作到 Triton](docs/user_guide/custom_operations_cn.md)
- 使用[模型集成](docs/user_guide/architecture_cn.md#ensemble-models)和[业务逻辑脚本(BLS)](https://github.com/triton-inference-server/python_backend#business-logic-scripting)启用模型流水线
- 通过设置[调度和批处理](docs/user_guide/architecture_cn.md#models-and-schedulers)参数和[模型实例](docs/user_guide/model_configuration_cn.md#instance-groups)优化您的模型
- 使用[模型分析器工具](https://github.com/triton-inference-server/model_analyzer)通过分析帮助优化您的模型配置
- 了解如何[通过加载和卸载模型显式管理可用模型](docs/user_guide/model_management_cn.md)

#### 配置和使用 Triton 推理服务器

- 阅读[快速入门指南](docs/getting_started/quickstart_cn.md)在 GPU 和 CPU 上运行 Triton 推理服务器
- Triton 支持多个执行引擎，称为[后端](https://github.com/triton-inference-server/backend#where-can-i-find-all-the-backends-that-are-available-for-triton)，包括 [TensorRT](https://github.com/triton-inference-server/tensorrt_backend)、[TensorFlow](https://github.com/triton-inference-server/tensorflow_backend)、[PyTorch](https://github.com/triton-inference-server/pytorch_backend)、[ONNX](https://github.com/triton-inference-server/onnxruntime_backend)、[OpenVINO](https://github.com/triton-inference-server/openvino_backend)、[Python](https://github.com/triton-inference-server/python_backend)等
- 并非所有上述后端都在 Triton 支持的每个平台上都受支持。查看[后端-平台支持矩阵](https://github.com/triton-inference-server/backend/blob/r24.10/docs/backend_platform_support_matrix.md)了解您的目标平台支持哪些后端
- 了解如何使用[性能分析器](https://github.com/triton-inference-server/perf_analyzer/blob/r24.10/README.md)和[模型分析器](https://github.com/triton-inference-server/model_analyzer)[优化性能](docs/user_guide/optimization_cn.md)
- 了解如何在 Triton 中[管理模型的加载和卸载](docs/user_guide/model_management_cn.md)
- 使用 [HTTP/REST JSON 或 gRPC 协议](docs/customization_guide/inference_protocols_cn.md#httprest-and-grpc-protocols)直接向 Triton 发送请求

#### 客户端支持和示例

Triton *客户端*应用程序向 Triton 发送推理和其他请求。[Python 和 C