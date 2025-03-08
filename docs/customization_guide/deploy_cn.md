<!--
# Copyright (c) 2020-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 安全部署注意事项

Triton 推理服务器项目设计灵活，允许开发人员以多种方式创建和部署推理解决方案。开发人员可以将 Triton 部署为 HTTP 服务器、gRPC 服务器、同时支持两者的服务器，或将 Triton 服务器嵌入到自己的应用程序中。开发人员可以在本地或云端部署 Triton，可以在 API 网关后的 Kubernetes 集群中部署，也可以作为独立进程部署。本指南旨在提供一些关键点和最佳实践，供部署基于 Triton 的解决方案的用户参考。

| [在安全网关或代理后部署](#deploying-behind-a-secure-proxy-or-gateway) | [以最小权限运行](#running-with-least-privilege) |

> [!IMPORTANT]
> 最终，基于 Triton 的解决方案的安全性是构建和部署该解决方案的开发人员的责任。在生产环境中部署时，请让安全专家审查任何潜在的风险和威胁。

> [!WARNING]
> 默认情况下，模型仓库的动态更新是禁用的。通过模型加载 API 或目录轮询启用模型仓库的动态更新可能导致任意代码执行。在生产部署中，模型仓库访问控制至关重要。如果需要动态更新，请确保只有受信任的实体才能访问模型加载 API 和模型仓库目录。

## 在安全代理或网关后部署

Triton 推理服务器主要设计为微服务，作为应用程序框架或服务网格中更大解决方案的一部分进行部署。

在这种部署中，通常使用专用的网关或代理服务器来处理授权、访问控制、资源管理、加密、负载均衡、冗余以及许多其他安全和可用性功能。

此类系统的完整设计超出了本部署指南的范围，但在这种情况下，专用入口控制器处理来自不受信任网络的访问，而 Triton 推理服务器仅处理受信任的、经过验证的请求。

在这种情况下，Triton 推理服务器不会直接暴露给不受信任的网络。

### 安全部署参考

在以下参考中，Triton 推理服务器将作为受信任内部网络中的"应用程序