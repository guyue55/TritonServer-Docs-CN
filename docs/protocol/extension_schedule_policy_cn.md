<!--
# Copyright 2020-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 调度策略扩展

本文档描述了 Triton 的调度策略扩展。调度策略扩展允许推理请求提供参数来影响 Triton 如何处理和调度请求。由于支持此扩展，Triton 在其服务器元数据的扩展字段中报告 "schedule_policy"。请注意，这些策略专门用于[动态批处理](https://github.com/triton-inference-server/server/blob/main/docs/user_guide/model_configuration_cn.md#dynamic-batcher)，并且仅对使用[直接](https://github.com/triton-inference-server/server/blob/main/docs/user_guide/architecture_cn.md#direct)调度策略的[序列批处理](https://github.com/triton-inference-server/server/blob/main/docs/user_guide/model_configuration_cn.md#sequence-batcher)提供实验性支持。

## 动态批处理

调度策略扩展使用请求参数来指示策略。参数及其类型为：

- "priority"：int64 值，表示请求的优先级。优先级值为零表示应使用默认优先级（即与不指定优先级参数的行为相同）。较低的优先级值表示较高的优先级。因此，最高优先级通过将参数设置为 1 来表示，次高优先级为 2，依此类推。

- "timeout"：int64 值，表示请求的超时值，以微秒为单位。如果请求无法在时间内完成，Triton 将采取特定于模型的操作，例如终止请求。

这两个参数都是可选的，如果未指定，Triton 将使用适合该模型的默认优先级和超时值来处理请求。

## 使用直接调度策略的序列批处理

**请注意，序列批处理的调度策略目前处于实验阶段，可能会发生变化。**

调度策略扩展使用请求参数来指示策略。参数及其类型为：

- "timeout"：int64 值，表示请求的超时值，以微秒为单位。如果请求无法在时间内完成，Triton 将终止请求以及相应的序列和该序列的已接收请求。超时将仅应用于尚未分配批处理槽位执行的序列的请求，已分配批处理槽位的序列的请求将不受超时设置的影响。

该参数是可选的，如果未指定，Triton 将根据模型配置处理请求和相应的序列。