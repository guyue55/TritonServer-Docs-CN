<!--
# Copyright (c) 2020, NVIDIA CORPORATION. All rights reserved.
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

# 分类扩展

本文档描述了 Triton 的分类扩展。分类扩展允许 Triton 将输出作为分类索引和（可选）标签返回，而不是将输出作为原始张量数据返回。由于支持此扩展，Triton 在其服务器元数据的 extensions 字段中报告 "classification"。

推理请求可以使用 "classification" 参数来请求返回一个或多个输出的分类结果。对于这样的输出，返回的张量不会是模型产生的形状和类型，而是类型为 BYTES，形状为 [ batch-size, <count> ]，其中每个元素将分类索引和标签作为单个字符串返回。返回张量的 <count> 维度将等于分类参数中指定的 "count" 值。

当使用分类参数时，Triton 将使用输出张量的数据类型比较，确定 n 个最高值元素作为 top-n 分类。例如，如果输出张量是 [ 1, 5, 10, 4 ]，最高值元素是 10（索引 2），其次是 5（索引 1），然后是 4（索引 3），最后是 1（索引 0）。因此，例如，按索引排序的前 2 个分类是 [ 2, 1 ]。

返回的字符串格式将是 "<value>:<index>[:<label>]"，其中 <index> 是类别在模型输出张量中的索引，<value> 是与该索引关联的模型输出中的值，而与该索引关联的 <label> 是可选的。例如，继续上面的例子，返回的张量将是 [ "10:2", "5:1" ]。如果模型有与这些索引关联的标签，返回的张量将是 [ "10:2:apple", "5:1:pickle" ]。

## HTTP/REST

在本文档中显示的所有 JSON 模式中，`$number`、`$string`、`$boolean`、`$object` 和 `$array` 指的是基本 JSON 类型。#optional 表示可选的 JSON 字段。

分类扩展要求 Triton 按照以下方式识别应用于请求的推理输出的 "classification" 参数：

- "classification"：`$number` 表示应该为输出返回的类别数量。

以下示例展示了如何在推理请求中使用分类参数。

```
POST /v2/models/mymodel/infer HTTP/1.1
Host: localhost:8000
Content-Type: application/json
Content-Length: <xx>
{
  "id" : "42",
  "inputs" : [
    {
      "name" : "input0",
      "shape" : [ 2, 2 ],
      "datatype" : "UINT32",
      "data" : [ 1, 2, 3, 4 ]
    }
  ],
  "outputs" : [
    {
      "name" : "output0",
      "parameters" : { "classification" : 2 }
    }
  ]
}
```