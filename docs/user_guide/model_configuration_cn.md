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

# 模型配置

**这是您第一次编写配置文件吗？** 请查看[本指南](https://github.com/triton-inference-server/tutorials/tree/main/Conceptual_Guide/Part_1-model_deployment#model-configuration)或此[示例](https://github.com/triton-inference-server/tutorials/tree/main/HuggingFace#examples)！

[模型仓库](model_repository.md)中的每个模型都必须包含一个模型配置，该配置提供了关于模型的必需和可选信息。通常，这个配置是在 config.pbtxt 文件中以 [ModelConfig protobuf](https://github.com/triton-inference-server/common/blob/main/protobuf/model_config.proto) 格式提供的。在某些情况下（在[自动生成的模型配置](#auto-generated-model-configuration)中讨论），模型配置可以由 Triton 自动生成，因此不需要显式提供。

本节描述了最重要的模型配置属性，但也应参考 [ModelConfig protobuf](https://github.com/triton-inference-server/common/blob/main/protobuf/model_config.proto) 中的文档。

## 最小模型配置

最小模型配置必须指定 [*platform* 和/或 *backend* 属性](https://github.com/triton-inference-server/backend/blob/main/README.md#backends)、*max_batch_size* 属性以及模型的输入和输出张量。

例如，考虑一个具有两个输入（*input0* 和 *input1*）和一个输出（*output0*）的 TensorRT 模型，所有这些都是 16 个条目的 float32 张量。最小配置如下：

```
  platform: "tensorrt_plan"
  max_batch_size: 8
  input [
    {
      name: "input0"
      data_type: TYPE_FP32
      dims: [ 16 ]
    },
    {
      name: "input1"
      data_type: TYPE_FP32
      dims: [ 16 ]
    }
  ]
  output [
    {
      name: "output0"
      data_type: TYPE_FP32
      dims: [ 16 ]
    }
  ]
```

### 名称、平台和后端

模型配置的 *name* 属性是可选的。如果在配置中未指定模型的名称，则假定它与包含该模型的模型仓库目录名称相同。如果指定了 *name*，它必须与包含该模型的模型仓库目录的名称匹配。*platform* 和 *backend* 的必需值在[后端文档](https://github.com/triton-inference-server/backend/blob/main/README.md#backends)中有描述。

### 模型事务策略

*model_transaction_policy* 属性描述了从模型中预期的事务性质。

#### 解耦

这个布尔设置指示模型生成的响应是否与发送给它的请求[解耦](./decoupled_models.md)。使用解耦意味着模型生成的响应数量可能与发出的请求数量不同，并且响应可能相对于请求的顺序而言是无序的。默认值为 false，这意味着模型将为每个请求精确生成一个响应。

### 最大批处理大小

*max_batch_size* 属性指示模型支持的最大批处理大小，用于 Triton 可以利用的[批处理类型](architecture.md#models-and-schedulers)。如果模型的批处理维度是第一个维度，并且模型的所有输入和输出都具有这个批处理维度，那么 Triton 可以使用其[动态批处理器](#dynamic-batcher)或[序列批处理器](#sequence-batcher)来自动对模型进行批处理。在这种情况下，*max_batch_size* 应设置为大于等于 1 的值，该值指示 Triton 应该与模型一起使用的最大批处理大小。

对于不支持批处理或不支持上述特定方式批处理的模型，*max_batch_size* 必须设置为零。