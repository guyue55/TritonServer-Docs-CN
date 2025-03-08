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

# 常见问题

## 与直接使用模型的框架 API 相比，使用 Triton Inference Server 运行模型有什么优势？

使用 Triton Inference Server 时，推理结果将与直接使用模型的框架相同。但是，使用 Triton 可以获得诸如[并发模型执行](architecture.md#concurrent-model-execution)（能够在同一 GPU 上同时运行多个模型）和[动态批处理](model_configuration.md#dynamic-batcher)以获得更好吞吐量等好处。您还可以在 Triton 和客户端应用程序运行时[替换或升级模型](model_management.md)。另一个好处是 Triton 可以作为 Docker 容器部署在任何地方 - 本地和公共云上。Triton Inference Server 还[支持多个框架](https://github.com/triton-inference-server/backend)，如 TensorRT、TensorFlow、PyTorch 和 ONNX，可在 GPU 和 CPU 上运行，从而实现简化的部署。

## Triton Inference Server 能在没有 GPU 的系统上运行吗？

是的，快速入门指南描述了如何[在仅 CPU 的系统上运行 Triton](../getting_started/quickstart.md#run-on-cpu-only-system)。

## Triton Inference Server 可以在非 Docker 环境中使用吗？

是的。Triton Inference Server 也可以在您的"裸机"系统上[从源代码构建](../customization_guide/build.md#building-without-docker)。

## 您是否提供 C++ 和 Python 以外的语言的客户端库？

我们提供 C++ 和 Python 客户端库，以便用户轻松编写与 Triton 通信的客户端应用程序。我们选择这些语言是因为它们可能在 ML 推理领域受欢迎且性能良好，但如果有需要，我们将来可能会添加其他语言。

我们提供 GRPC API 作为为大量语言生成自己的客户端库的方式。通过遵循官方 GRPC 文档并使用 [grpc_service.proto](https://github.com/triton-inference-server/common/blob/main/protobuf/grpc_service.proto)，您可以为 GRPC 支持的所有语言生成语言绑定。我们为 [Go](https://github.com/triton-inference-server/client/blob/main/src/grpc_generated/go)、[Python](https://github.com/triton-inference-server/client/blob/main/src/python/examples/grpc_client.py) 和 [Java](https://github.com/triton-inference-server/client/blob/main/src/grpc_generated/java) 提供了三个示例。

一般来说，客户端库（和客户端示例）就是示例。我们认为客户端库编写良好且经过良好测试，但它们并不意味着要服务于每个可能的用例。在某些情况下，您可能希望开发自己的定制库以满足您的特定需求。

## 如何在 AWS 环境中使用 Triton Inference Server？

在 AWS 环境中，Triton Inference Server docker 容器可以在[仅 CPU 实例或 GPU 计算实例](../getting_started/quickstart.md#launch-triton)上运行。Triton 可以直接在计算实例上运行，也可以在 Elastic Kubernetes Service (EKS) 内运行。此外，其他 AWS 服务（如 Elastic Load Balancer (ELB)）可用于在多个 Triton 实例之间进行负载均衡。Elastic Block Store (EBS) 或 S3 可用于存储推理服务器加载的深度学习模型。

## 如何测量在 Triton Inference Server 中运行的模型的性能？

Triton Inference Server 通过两种方式公开性能信息：通过 [Prometheus 指标](metrics.md)和通过 [HTTP/REST、GRPC 和 C API](../customization_guide/inference_protocols.md) 提供的统计信息。

客户端应用程序 [perf_analyzer](https://github.com/triton-inference-server/perf_analyzer/blob/main/README.md) 允许您使用合成负载测量单个模型的性能。perf_analyzer 应用程序旨在向您展示延迟与吞吐量之间的权衡。

## 如何使用 Triton Inference Server 充分利用 GPU？

Triton Inference Server 具有几个旨在提高 GPU 利用率的功能：

* Triton 可以[同时为多个模型执行推理](architecture.md#concurrent-model-execution)（使用相同或不同的框架）使用同一 GPU。

* Triton 可以通过使用[同一模型的多个实例](architecture.md#concurrent-model-execution)来处理对该模型的多个同时推理请求，从而提高推理吞吐量。Triton 选择合理的默认值，但[您也可以在每个模型的基础上控制确切的并发级别](model_configuration.md#instance-groups)。

* Triton 可以[将多个推理请求批处理到单个推理执行中](model_configuration.md#dynamic-batcher)。通常，批处理推理请求会导致吞吐量大幅提高，而延迟只会相对较小地增加。

作为一般规则，批处理是提高 GPU 利用率最有益的方式。因此，您应该始终尝试为您的模型启用[动态批处理器](model_configuration.md#dynamic-batcher)。使用模型的多个实例也可以提供一些好处，但通常对计算需求较小的模型最有用。大多数模型使用两个实例会受益，但超过这个数量通常没有用处。

## 如果我有一个具有多个 GPU 的服务器，我应该使用一个 Triton Inference Server 来管理所有 GPU，还是应该使用多个推理服务器，每个 GPU 一个？

Triton Inference Server 将利用服务器上它可以访问的所有 GPU。您可以通过使用 CUDA_VISIBLE_DEVICES 环境变量（对于 Docker，您也可以使用 NVIDIA_VISIBLE_DEVICES 或在启动容器时使用 --gpus 标志）来限制 Triton 可用的 GPU。在使用多个 GPU 时，Triton 将在 GPU 之间分配推理请求，以保持它们都得到同等利用。您还可以[更明确地控制哪些模型在哪些 GPU 上运行](model_configuration.md#instance-groups)。

在某些部署和编排环境（例如，Kubernetes）中，可能更希望将单个多 GPU 服务器分区为多个*节点*，每个节点有一个 GPU。在这种情况下，编排环境将为每个 GPU 运行不同的 Triton，并使用负载均衡器在可用的 Triton 实例之间分配推理请求。

## 如果服务器出现段错误，如何调试？

NGC 构建是一个 Release 构建，不包含调试符号。build.py 默认也是 Release 构建。请参考[build.md](../customization_guide/build.md#building-with-debug-symbols)中的说明创建 Triton 的 Debug 构建。这将有助于在查看段错误的 gdb 跟踪时找到原因。

在为 Triton 的段错误打开 GitHub issue 时，请包含回溯以帮助我们更好地解决问题。

## 作为 [NVIDIA AI Enterprise Software Suite](https://www.nvidia.com/en-us/data-center/products/ai-enterprise/) 的一部分使用 [Triton Inference Server](https://developer.nvidia.com/triton-inference-server) 有什么好处？

NVIDIA AI Enterprise 通过提供完整的端到端 AI 平台，使企业能够实现完整的 AI 工作流程。四个主要好处：

### 企业级支持、安全性和 API 稳定性：

通过 NVIDIA Enterprise Support，业务关键型 AI 项目保持正轨，该支持在全球范围内提供，可帮助 IT 团队部署和管理 AI 应用程序的生命周期，并帮助开发团队构建 AI 应用程序。支持包括维护更新、可靠的 SLA 和响应时间。定期的安全审查和优先通知降低了未管理开源的潜在风险，并确保符合企业标准。最后，长期支持和回归测试确保版本之间的 API 稳定性。

### 通过 AI 工作流和预训练模型加快生产时间：
NVIDIA AI Enterprise 包含 [AI 工作流](https://www.nvidia.com/en-us/launchpad/ai/workflows/)，以减少开发常见 AI 应用程序的复杂性，这些工作流是针对特定业务成果的参考应用程序，如智能虚拟助手和用于实时网络安全威胁检测的数字指纹。AI 工作流参考应用程序可能包括 [AI 框架](https://docs.nvidia.com/deeplearning/frameworks/index.html)和[预训练模型](https://developer.nvidia.com/ai-models)、[Helm Charts](https://catalog.ngc.nvidia.com/helm-charts)、[Jupyter Notebooks](https://developer.nvidia.com/run-jupyter-notebooks) 和[文档](https://docs.nvidia.com/ai-enterprise/index.html#overview)。

### 提高效率和节省成本的性能：
将加速计算用于 AI 工作负载，如使用 [NVIDIA RAPIDS Accelerator](https://developer.nvidia.com/rapids) 进行 Apache Spark 数据处理和使用 Triton Inference Server 进行推理，可提供更好的性能，这也提高了效率并降低了运营和基础设施成本，包括减少时间和能源消耗带来的节省。