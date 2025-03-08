<!--
# Copyright (c) 2021, NVIDIA CORPORATION. All rights reserved.
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

# 仓库代理

*仓库代理*通过在模型加载或卸载时执行的新功能扩展 Triton。您可以引入自己的代码来执行身份验证、解密、转换或在模型加载时的类似操作。

**测试版：仓库代理 API 是测试版质量，在一个或多个发布版本中可能会有不向后兼容的更改。**

仓库代理使用[仓库代理 API](https://github.com/triton-inference-server/core/tree/main/include/triton/core/tritonrepoagent.h)与 Triton 通信。[checksum_repository_agent GitHub 仓库](https://github.com/triton-inference-server/checksum_repository_agent)提供了一个仓库代理示例，该代理在加载模型之前验证文件校验和。

## 使用仓库代理

模型可以通过在[模型配置](../user_guide/model_configuration.md)的 *ModelRepositoryAgents* 部分中指定一个或多个仓库代理来使用它们。每个仓库代理可以在模型配置中指定特定于该代理的参数，以控制代理的行为。要了解给定代理可用的参数，请查阅该代理的文档。

可以为同一个模型指定多个代理，它们将在模型加载或卸载时按顺序调用。以下示例模型配置内容显示了如何指定两个代理"agent0"和"agent1"，以便按照给定的参数按顺序调用它们。

```
model_repository_agents
{
  agents [
    {
      name: "agent0",
      parameters [
        {
          key: "key0",
          value: "value0"
        },
        {
          key: "key1",
          value: "value1"
        }
      ]
    },
    {
      name: "agent1",
      parameters [
        {
          key: "keyx",
          value: "valuex"
        }
      ]
    }
  ]
}
```

## 实现仓库代理

仓库代理必须实现为共享库，共享库的名称必须为 *libtritonrepoagent_\<repo-agent-name\>.so*。共享库应隐藏除仓库代理 API 所需的符号之外的所有符号。有关如何使用 ldscript 仅公开必要符号的示例，请参阅[校验和示例的 CMakeList.txt](https://github.com/triton-inference-server/checksum_repository_agent/blob/main/CMakeLists.txt)。