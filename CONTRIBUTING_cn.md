<!--
# Copyright 2018-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 贡献指南

修复文档错误或对现有代码进行小改动的贡献可以按照以下规则直接提交并提交适当的 PR。

旨在添加重要新功能的贡献必须遵循以下几点所述的更具协作性的路径。在提交添加重大增强或扩展的大型 PR 之前，请务必提交描述提议更改的 GitHub issue，以便 Triton 团队能够提供反馈。

- 作为 GitHub issue 讨论的一部分，将就您的更改达成设计共识。前期的设计讨论是必要的，以确保您的增强功能以符合 Triton 整体架构的方式实现。

- Triton 项目分布在多个代码仓库中。Triton 团队将提供关于如何以及在何处实现您的增强功能的指导。

- [测试](docs/customization_guide/test_cn.md)是任何 Triton 增强功能的关键部分。您应该计划在为您的更改创建测试上投入大量时间。Triton 团队将帮助您设计测试，使其与现有的测试基础设施兼容。

- 如果您的增强功能提供了用户可见的功能，那么您需要提供文档。

# 贡献规则

- 代码风格约定由 clang-format 强制执行。请参阅下文了解如何确保您的贡献符合规范。通常，在添加新代码或扩展/修复现有功能时，请遵循相关文件、子模块、模块和项目中的现有约定。

- 避免在现有代码中引入不必要的复杂性，以保持可维护性和可读性。

- 尽量保持拉取请求（PR）的简洁：

  - 避免提交注释掉的代码。

  - 在可能的情况下，每个 PR 应该只解决一个问题。如果有几个本来无关的问题需要修复才能达到预期目标，完全可以开启几个 PR，并在描述中说明哪个 PR 依赖于另一个 PR。单个 PR 中的更改越复杂，审查这些更改所需的时间就越长。

  - 确保构建日志是干净的，即不应该出现警告或错误。

- 确保所有 `L0_*` 测试通过：

  - 在 `qa/` 目录中，有一些基本的健全性测试脚本，位于名为 `L0_...` 的目录中。有关运行这些测试的说明，请参阅[测试](docs/customization_guide/test_cn.md)文档。

- Triton 推理服务器的默认构建假定使用最新版本的依赖项（CUDA、TensorFlow、PyTorch、TensorRT 等）。添加与这些依赖项的旧版本兼容性的贡献将被考虑，但 NVIDIA 不能保证所有可能的构建配置都能工作，不会被未来的贡献破坏，并保持最高性能。

- 确保您可以将您的工作贡献给开源（您的代码不会引入许可证和/或专利冲突）。在您的 PR 被合并之前，您需要完成下面描述的 CLA。

- 感谢您提前对我们审查您的贡献的耐心；我们确实很感谢它们！

# 编码约定

所有拉取请求都会根据[存储库顶级 .pre-commit-config.yaml](https://github.com/NVIDIA/triton-inference-server/blob/master/pre-commit-config.yaml) 中的 [pre-commit hooks](https://github.com/pre-commit/pre-commit-hooks) 进行检查。这些钩子会进行一些健全性检查，如代码检查和格式化。要合并更改，这些检查必须通过。

要在本地运行这些检查，您可以[安装 pre-commit](https://pre-commit.com/#install)，然后在克隆的仓库内运行 `pre-commit install`。当您提交更改时，pre-commit 钩子将自动运行。如果 pre-commit 钩子实现了修复，再次添加文件并运行 `git commit` 第二次将通过并成功提交。

# 贡献者许可协议（CLA）

Triton 要求所有贡献者（或其公司实体）将签署的[贡献者许可协议](https://github.com/NVIDIA/triton-inference-server/blob/master/Triton-CCLA-v1.pdf)副本发送至 triton-cla@nvidia.com。
*注意*：没有公司从属关系的贡献者可以在"公司名称"和"公司地址"字段中填写"N/A"。