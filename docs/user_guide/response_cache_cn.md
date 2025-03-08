<!--
# Copyright 2021-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# Triton 响应缓存

## 概述

在本文档中，*推理请求*是指提交给 Triton 的模型名称、模型版本和输入张量（名称、形状、数据类型和张量数据）。推理结果是由推理执行产生的输出张量（名称、形状、数据类型和张量数据）。响应缓存被 Triton 用来保存之前执行的推理请求生成的推理结果。Triton 将维护响应缓存，使得在缓存中命中的推理请求不需要执行模型来产生结果，而是从缓存中提取结果。对于某些用例，这可以显著降低推理请求的延迟。

Triton 使用包含模型名称、模型版本和模型输入的推理请求的哈希值来访问响应缓存。如果在缓存中找到哈希值，则从缓存中提取相应的推理结果并用于该请求。当这种情况发生时，Triton 不需要执行模型来产生推理结果。如果在缓存中找不到哈希值，Triton 执行模型来产生推理结果，然后将该结果记录在缓存中，以便后续的推理请求可以（重新）使用这些结果。

## 使用方法

要在给定模型上使用缓存，必须在服务器端和模型的[模型配置](model_configuration.md#response-cache)中都启用它。有关更多详细信息，请参见以下各节。

### 在服务器端启用缓存

通过在启动 Triton 服务器时指定缓存实现名称 `<cache>` 和相应的配置，在服务器端启用响应缓存。

在 CLI 中，这转换为设置 `tritonserver --cache-config <cache>,<key>=<value> ...`。例如：
```
tritonserver --cache-config local,size=1048576
```

> [!注意]
> 如果使用非交互式 shell，您可能需要在参数中不使用空格，如：`--cache-config=<cache>,<key>=<value>`。

对于进程内 C API 应用程序，这转换为调用 `TRITONSERVER_SetCacheConfig(const char* cache_implementation, const char* config_json)`。

这允许用户在服务器启动时全局启用/禁用缓存。

### 为模型启用缓存

**默认情况下，即使使用 `--cache-config` 标志在全局启用了响应缓存，任何模型也不会使用响应缓存。**

要让给定模型使用响应缓存，该模型还必须在其模型配置中启用响应缓存：
```
# config.pbtxt

response_cache {
  enable: true
}
```

这允许用户为特定模型启用/禁用缓存。

有关为每个模型启用响应缓存的更多信息，请参见[模型配置文档](model_configuration.md#response-cache)。

### 缓存实现

从 23.03 版本开始，Triton 有一组 [TRITONCACHE API](https://github.com/triton-inference-server/core/blob/main/include/triton/core/tritoncache.h)，用于与用户选择的缓存实现进行通信。

缓存实现是一个实现所需 TRITONCACHE API 的共享库，如果启用，则在服务器启动时动态加载。

Triton 最新的 [tritonserver 发布容器](https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tritonserver)自带以下缓存实现：
- [local](https://github.com/triton-inference-server/local_cache)：`/opt/tritonserver/caches/local/libtritoncache_local.so`
- [redis](https://github.com/triton-inference-server/redis_cache)：`/opt/tritonserver/caches/redis/libtritoncache_redis.so`

通过这些 TRITONCACHE API，`tritonserver` 公开了一个新的 `--cache-config` CLI 标志，让用户可以灵活地自定义要使用的缓存实现以及如何配置它。与 `--backend-config` 标志类似，预期格式为 `--cache-config <cache_name>,<key>=<value>`，如果缓存实现需要，可以多次指定以指定多个键。

#### 本地缓存

`local` 缓存实现等同于 23.03 版本之前在内部使用的响应缓存。有关更多实现细节，请参见[本地缓存实现](https://github.com/triton-inference-server/local_cache)。

当指定 `--cache-config local,size=SIZE` 且 `SIZE` 非零时，Triton 在 CPU 内存中分配请求的大小，并**在所有推理请求和所有模型之间共享缓存**。

#### Redis 缓存

`redis` 缓存实现使 Triton 能够与 Redis 服务器进行缓存通信。`redis_cache` 实现本质上是一个 Redis 客户端，充当 Triton 和 Redis 之间的中介。

在 Triton 上下文中，与 `local` 缓存相比，`redis` 缓存的一些好处包括：
- 只要 Triton 可以访问，Redis 服务器就可以远程托管，因此它不直接与 Triton 进程生命周期绑定。
  - 这意味着 Triton 可以重新启动，并且仍然可以访问以前缓存的条目。
  - 这也意味着 Triton 不必与缓存竞争内存/资源使用。
- 通过将每个 Triton 实例配置为与同一个 Redis 服务器通信，多个 Triton 实例可以共享缓存。
- Redis 服务器可以独立于 Triton 进行更新/重启，在 Redis 服务器停机期间，Triton 将回退到无缓存访问的操作方式，并记录适当的错误。

一般来说，Redis 服务器可以根据您的用例需求进行配置/部署，Triton 的 `redis` 缓存将简单地充当您的 Redis 部署的客户端。有关配置 Redis 服务器的问题和详细信息，应参考 [Redis 文档](https://redis.io/docs/)。

有关 Triton 特定的 `redis` 缓存实现详细信息/配置，请参见[redis 缓存实现](https://github.com/triton-inference-server/redis_cache)。

#### 自定义缓存

通过 TRITONCACHE API 接口，用户现在可以实现自己的缓存以满足任何特定用例需求。要查看缓存开发人员必须实现的所需接口，请参见 [TRITONCACHE API 头文件](https://github.com/triton-inference-server/core/blob/main/include/triton/core/tritoncache.h)。可以使用 `local` 或 `redis` 缓存实现作为参考。

成功开发和构建自定义缓存后，生成的共享库（例如：`libtritoncache_<name>.so`）必须放在缓存目录中，类似于 `local` 和 `redis` 缓存实现所在的位置。默认情况下，此目录是 `/opt/tritonserver/caches`，但可以根据需要使用 `--cache-dir` 指定自定义目录。

举个例子，如果自定义缓存名为"custom"（此名称是任意的），默认情况下，Triton 会期望在 `/opt/tritonserver/caches/custom/libtritoncache_custom.so` 找到缓存实现。

## 弃用说明

> **注意**
> 在 23.03 之前，启用 `local` 缓存是通过在启动 Triton 时使用 `--response-cache-byte-size` 标志设置非零大小（以字节为单位）来完成的。
>
> 从 23.03 开始，`--response-cache-byte-size` 标志现在已弃用，应改用 `--cache-config`。为了向后兼容，`--response-cache-byte-size` 将继续通过转换为相应的 `--cache-config` 参数在后台运行，但它将默认使用 `local` 缓存实现。使用 `--response-cache-byte-size` 标志无法选择其他缓存实现。
>
> 例如，`--response-cache-byte-size 1048576` 等同于 `--cache-config local,size=1048576`。但是，`--cache-config` 标志更加灵活，应该使用它。

> **警告**
>
> 对于非常小的 `--cache-config local,size=<small_value>` 或 `--response-cache-byte-size` 值（例如：小于 1024 字节），由于内部内存管理要求，`local` 缓存实现可能无法初始化。如果您遇到相对较小的缓存大小的初始化错误，请尝试增加它。
>
> 同样，大小受系统上可用 RAM 的限制。如果您遇到非常大的缓存大小设置的初始分配错误，请尝试减小它。

## 性能

响应缓存旨在用于预期有大量重复请求（缓存命中）的用例，因此会从缓存中受益。这里的"大量