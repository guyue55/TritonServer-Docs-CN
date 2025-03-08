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

# 速率限制器

速率限制器管理 Triton 在模型实例上调度请求的速率。速率限制器在 Triton 中加载的所有模型之间运行，以实现*跨模型优先级*。

在没有速率限制的情况下（--rate-limit=off），Triton 会在模型实例可用时立即调度请求（或使用动态批处理时的一组请求）的执行。这种行为通常最适合性能。但是，在某些情况下，同时运行所有模型可能会给服务器带来过大的负载。例如，某些框架上的模型执行会动态分配内存。同时运行所有这些模型可能会导致系统内存不足。

速率限制器允许推迟某些模型实例上的推理执行，使它们不会同时运行。模型优先级用于决定下一步调度哪个模型实例。

## 使用速率限制器

要启用速率限制，用户必须在启动 tritonserver 时设置 `--rate-limit` 选项。有关更多信息，请参阅 `tritonserver --help` 输出的选项用法。

速率限制器由每个模型实例的速率限制器配置控制，如[速率限制器配置](model_configuration.md#rate-limiter-configuration)中所述。速率限制器配置包括实例组定义的模型实例的[资源](model_configuration.md#resources)和[优先级](model_configuration.md#priority)。

### 资源

资源由唯一名称和表示资源副本数量的计数标识。默认情况下，模型实例不使用速率限制器资源。通过列出资源/计数，模型实例表明它需要在模型实例设备上有多少资源可用才能允许执行。在执行时，指定的多个资源被分配给模型实例，仅在执行结束时释放。默认情况下，可用的资源副本数量是列出该资源的所有模型实例中的最大值。例如，假设三个已加载的模型实例 A、B 和 C 分别为单个设备指定以下资源要求：

```
A: [R1: 4, R2: 4]
B: [R2: 5, R3: 10, R4: 5]
C: [R1: 1, R3: 7, R4: 2]
```

默认情况下，根据这些模型实例要求，服务器将创建具有以下副本数的资源：

```
R1: 4
R2: 5
R3: 10
R4: 5
```

这些值确保所有模型实例都可以成功调度。可以使用 `--rate-limit-resource` 选项在命令行上明确给出资源的默认值。`tritonserver --help` 将提供更详细的使用说明。

默认情况下，可用的资源副本是按设备分配的，模型实例的资源要求是根据运行模型实例的设备相关联的资源强制执行的。`--rate-limit-resource` 允许用户为不同设备提供不同的资源副本。速率限制器还可以处理全局资源。全局资源不是在每个设备上创建资源副本，而是在整个系统中只有一个副本。

速率限制器依赖于模型配置来确定资源是全局的还是非全局的。有关如何在模型配置中指定它们的更多详细信息，请参见[资源](model_configuration.md#resources)。

对于在双设备机器上运行的 tritonserver，使用 `--rate-limit-resource=R1:10 --rate-limit-resource=R2:5:0 --rate-limit-resource=R2:8:1 --rate-limit-resource=R3:2` 调用时，可用的资源副本为：

```
全局   => [R3: 2]
设备 0 => [R1: 10, R2: 5]
设备 1 => [R1: 10, R2: 8]
```

其中 R3 在某个已加载的模型中显示为全局资源。

### 优先级

在资源受限的系统中，模型实例之间会为执行其推理请求而争用资源。优先级设置有助于确定选择哪个模型实例进行下一次执行。有关更多信息，请参见[优先级](model_configuration.md#priority)。