<!--
# Copyright (c) 2021, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 不规则批处理

Triton 提供[动态批处理功能](model_configuration.md#dynamic-batcher)，可以将同一模型执行的多个请求组合在一起，以提供更高的吞吐量。默认情况下，只有当每个请求中的每个输入具有相同的形状时，才能进行动态批处理。为了在输入形状经常变化的情况下利用动态批处理，客户端需要将请求中的输入张量填充到相同的形状。

不规则批处理是一个避免显式填充的功能，它允许用户指定哪些输入不需要形状检查。用户可以通过在模型配置中设置 `allow_ragged_batch` 字段来指定这样的输入（不规则输入）：

```
...
input [
  {
    name: "input0"
    data_type: TYPE_FP32
    dims: [ 16 ]
    allow_ragged_batch: true
  }
]
...
```

不规则输入在一批请求中的处理方式取决于后端实现。诸如 [ONNX Runtime 后端](https://github.com/triton-inference-server/onnxruntime_backend)、[TensorFlow 后端](https://github.com/triton-inference-server/tensorflow_backend)、[PyTorch 后端](https://github.com/triton-inference-server/pytorch_backend)和 [TensorRT 后端](https://github.com/triton-inference-server/tensorrt_backend)等后端要求模型接受一维张量作为不规则输入。这些后端将请求输入连接成一维张量。

由于连接后的输入无法跟踪每个请求的起始和结束索引，后端通常要求模型具有额外的输入（[批处理输入](#batch-input)）来描述所形成批处理的各种信息。

## 批处理输入

批处理输入通常与不规则输入结合使用，以提供有关每个批处理元素的信息，例如批处理中每个请求的输入元素计数。批处理输入是由 Triton 生成的，而不是在请求中提供的，因为这些信息只能在动态批处理形成后才能确定。

除了元素计数外，用户还可以指定其他类型的批处理输入，详细信息请参见[protobuf 文档](https://github.com/triton-inference-server/common/blob/main/protobuf/model_config.proto)。

## 不规则输入和批处理输入示例

如果您有一个接受 1 个可变长度输入张量 INPUT 的模型，其形状为 [ -1, -1 ]。第一个维度是批处理维度，第二个维度是可变长度内容。当客户端发送 3 个形状分别为 [ 1, 3 ]、[ 1, 4 ]、[ 1, 5 ] 的请求时。要利用动态批处理，实现此模型的直接方法是期望 INPUT 形状为 [ -1, -1 ]，并假设所有输入都填充到相同的长度，使所有请求变为形状 [ 1, 5 ]，这样 Triton 就可以将它们批处理并作为单个 [ 3, 5 ] 张量发送给模型。在这种情况下，填充张量和对填充内容进行额外的模型计算会产生开销。
以下是输入配置：

```
max_batch_size: 16
input [
  {
    name: "INPUT"
    data_type: TYPE_FP32
    dims: [ -1 ]
  }
]
```

使用 Triton 不规则批处理，模型将被实现为期望 INPUT 形状为 [ -1 ] 并具有一个额外的批处理输入 INDEX，形状为 [ -1 ]，模型应使用它来解释 INPUT 中的批处理元素。对于这样的模型，客户端请求不需要填充，可以按原样发送（形状为 [ 1, 3 ]、[ 1, 4 ]、[ 1, 5 ]）。上述后端将输入批处理成形状为 [ 12 ] 的张量，其中包含请求的 3 + 4 + 5 连接。Triton 还创建形状为 [ 3 ] 的批处理输入张量，其值为 [ 3, 7, 12 ]，给出了每个批处理元素在输入张量中结束的偏移量。以下是输入配置：

```
max_batch_size: 16
input [
  {
    name: "INPUT"
    data_type: TYPE_FP32
    dims: [ -1 ]
    allow_ragged_batch: true
  }
]
batch_input [
  {
    kind: BATCH_ACCUMULATED_ELEMENT_COUNT
    target_name: "INDEX"
    data_type: TYPE_FP32
    source_input: "INPUT"
  }
]
```

上述示例使用了 [`BATCH_ACCUMULATED_ELEMENT_COUNT`](https://github.com/triton-inference-server/common/blob/main/protobuf/model_config.proto) 类型的不规则批处理。[protobuf 文档](https://github.com/triton-inference-server/common/blob/main/protobuf/model_config.proto)中描述的其他类型的操作方式类似。