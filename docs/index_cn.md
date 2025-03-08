<!--
# Copyright 2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

::::{grid}
:reverse:
:gutter: 2 1 1 1
:margin: 4 4 1 1

:::{grid-item}
:columns: 4

```{image} ./_static/nvidia-logo-vert-rgb-blk-for-screen.png
:width: 300px
```
:::
:::{grid-item}
:columns: 8
:class: sd-fs-3

NVIDIA Triton 推理服务器

:::
::::

Triton 推理服务器是一个开源的推理服务软件，可以简化 AI 推理过程。

  <!-- :::
  :align: center
  [![Getting Started Video](https://img.youtube.com/vi/NQDtfSi5QF4/1.jpg)](https://www.youtube.com/watch?v=NQDtfSi5QF4)
  ::: -->

<div>
<iframe width="560" height="315" src="https://www.youtube.com/embed/NQDtfSi5QF4" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</div>

# Triton 推理服务器

Triton 推理服务器使团队能够部署来自多个深度学习和机器学习框架的任何 AI 模型，包括 TensorRT、TensorFlow、PyTorch、ONNX、OpenVINO、Python、RAPIDS FIL 等。Triton 支持在 NVIDIA GPU、x86 和 ARM CPU 或 AWS Inferentia 上跨云、数据中心、边缘和嵌入式设备进行推理。Triton 推理服务器为多种查询类型提供优化的性能，包括实时、批处理、集成和音视频流。Triton 推理服务器是 [NVIDIA AI Enterprise](https://www.nvidia.com/en-us/data-center/products/ai-enterprise/) 的一部分，这是一个加速数据科学流程并简化生产 AI 开发和部署的软件平台。

主要特点包括：

- [支持多个深度学习框架](https://github.com/triton-inference-server/backend#where-can-i-find-all-the-backends-that-are-available-for-triton)
- [支持多个机器学习框架](https://github.com/triton-inference-server/fil_backend)
- [并发模型执行](user_guide/architecture_cn.md#concurrent-model-execution)
- [动态批处理](user_guide/model_configuration_cn.md#dynamic-batcher)
- [序列批处理](user_guide/model_configuration_cn.md#sequence-batcher)和[隐式状态管理](user_guide/architecture_cn.md#implicit-state-management)用于有状态模型
- 提供[后端 API](https://github.com/triton-inference-server/backend)，允许添加自定义后端和预/后处理操作
- 使用[集成](user_guide/architecture_cn.md#ensemble-models)或[业务逻辑脚本(BLS)](https://github.com/triton-inference-server/python_backend#business-logic-scripting)的模型流水线
- 基于社区开发的[KServe 协议](https://github.com/kserve/kserve/tree/master/docs/predict-api/v2)的[HTTP/REST 和 GRPC 推理协议](customization_guide/inference_protocols_cn.md)
- [C API](customization_guide/inference_protocols.md#in-process-triton-server-api)和[Java API](customization_guide/inference_protocols.md#java-bindings-for-in-process-triton-server-api)允许 Triton 直接链接到您的应用程序中，用于边缘和其他进程内用例
- [指标](user_guide/metrics_cn.md)显示 GPU 利用率、服务器吞吐量、服务器延迟等

加入 [Triton 和 TensorRT 社区](https://www.nvidia.com/en-us/deep-learning-ai/triton-tensorrt-newsletter/)，及时了解最新的产品更新、错误修复、内容、最佳实践等。需要企业支持？通过 [NVIDIA AI Enterprise 软件套件](https://www.nvidia.com/en-us/data-center/products/ai-enterprise/)，NVIDIA 全球支持可用于 Triton 推理服务器。

查看[最新发行说明](https://docs.nvidia.com/deeplearning/triton-inference-server/release-notes/rel-23-05.html#rel-23-05)了解最新功能和错误修复。