<!--
# Copyright 2021-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# Triton 推理服务器对 Jetson 和 JetPack 的支持

在[发行说明](https://github.com/triton-inference-server/server/releases)的附加 tar 文件中提供了用于 [JetPack 5.0](https://developer.nvidia.com/embedded/jetpack) 的 Triton 版本。

![Triton 在 Jetson 上的架构图](images/triton_on_jetson.png)

Triton 推理服务器在 JetPack 上的支持包括：

* 在 GPU 和 NVDLA 上运行模型
* [并发模型执行](architecture_cn.md#concurrent-model-execution)
* [动态批处理](architecture_cn.md#models-and-schedulers)
* [模型流水线](architecture_cn.md#ensemble-models)
* [可扩展后端](https://github.com/triton-inference-server/backend)
* [HTTP/REST 和 GRPC 推理协议](../customization_guide/inference_protocols.md)
* [C API](../customization_guide/inference_protocols.md#in-process-triton-server-api)

JetPack 5.0 的限制：

* Onnx Runtime 后端不支持 OpenVino 和 TensorRT 执行提供程序。CUDA 执行提供程序处于 Beta 阶段。
* Python 后端不支持 GPU 张量和异步 BLS。
* 不支持 CUDA IPC（共享内存）。但支持系统共享内存。
* 不支持 GPU 指标、GCS 存储、S3 存储和 Azure 存储。

在 JetPack 上，虽然支持 HTTP/REST 和 GRPC 推理协议，但对于边缘用例，建议使用直接的 [C API 集成](../customization_guide/inference_protocols.md#in-process-triton-server-api)。

您可以从 Triton 推理服务器的[发布页面](https://github.com/triton-inference-server/server/releases)的 _"Jetson JetPack Support"_ 部分下载用于 Jetson 的 `.tgz` 文件。

`.tgz` 文件包含 Triton 服务器可执行文件和共享库，以及 C++ 和 Python 客户端库和示例。

## 安装和使用

### Triton 的构建依赖

在构建 Triton 服务器之前必须安装以下依赖：

```
apt-get update && \
        apt-get install -y --no-install-recommends \
            software-properties-common \
            autoconf \
            automake \
            build-essential \
            git \
            libb64-dev \
            libre2-dev \
            libssl-dev \
            libtool \
            libboost-dev \
            rapidjson-dev \
            patchelf \
            pkg-config \
            libopenblas-dev \
            libarchive-dev \
            zlib1g-dev \
            python3 \
            python3-dev \
            python3-pip
```

必须安装额外的 Onnx Runtime 依赖项才能构建 Onnx Runtime 后端：

```
pip3 install --upgrade flake8 flatbuffers
```

必须安装额外的 PyTorch 依赖项才能构建（和运行）PyTorch 后端：

```
apt-get -y install autoconf \
            bc \
            g++-8 \
            gcc-8 \
            clang-8 \
            lld-8

pip3 install --upgrade expecttest xmlrunner hypothesis aiohttp pyyaml scipy ninja typing_extensions protobuf
```

除了这些 PyTorch 依赖项外，还必须安装与发布版本对应的 PyTorch wheel（用于构建和运行时）：

```
pip3 install --upgrade https://developer.download.nvidia.com/compute/redist/jp/v50/pytorch/torch-1.12.0a0+2c916ef.nv22.3-cp38-cp38-linux_aarch64.whl
```

在构建 Triton 客户端库/示例之前必须安装以下依赖项：

```
apt-get install -y --no-install-recommends \
            curl \
            jq

pip3 install --upgrade wheel setuptools cython && \
    pip3 install --upgrade grpcio-tools "numpy<2" attrdict pillow
```

**注意**：OpenCV 4.2.0 作为 JetPack 的一部分安装。它是客户端构建的依赖项之一。

**注意**：在 Jetson 上构建 Triton 时，您需要较新版本的 cmake。我们建议使用 cmake 3.25.2。以下是将 cmake 版本升级到 3.25.2 的脚本。

```
apt remove cmake
# 使用来自 https://apt.kitware.com/ 的 CMAKE 安装说明
apt update && apt install -y gpg wget && \
      wget -O - https://apt.kitware.com/keys/kitware-archive-latest.asc 2>/dev/null | \
            gpg --dearmor - |  \
            tee /usr/share/keyrings/kitware-archive-keyring.gpg >/dev/null && \
      . /etc/os-release && \
      echo "deb [signed-by=/usr/share/keyrings/kitware-archive-keyring.gpg] https://apt.kitware.com/ubuntu/ $UBUNTU_CODENAME main" | \
      tee /etc/apt/sources.list.d/kitware.list >/dev/null && \
      apt-get update && \
      apt-get install -y --no-install-recommends cmake cmake-data
```

### Triton 的运行时依赖

在运行 Triton 服务器之前必须安装以下运行时依赖项：

```
apt-get update && \
        apt-get install -y --no-install-recommends \
        libb64-0d \
        libre2-9 \
        libssl1.1 \
        rapidjson-dev \
        libopenblas-dev \
        libarchive-dev \
        zlib1g \
        python3 \
        python3-dev \
        python3-pip
```

在运行 Triton 客户端之前必须安装以下运行时依赖项：

```
apt-get update && \
        apt-get install -y --no-install-recommends \
        curl \
        jq

pip3 install --upgrade wheel setuptools && \
    pip3 install --upgrade grpcio-tools "numpy<2" attrdict pillow
```

PyTorch 运行时依赖项与上面列出的构建依赖项相同。

### 使用

**注意**：PyTorch 后端依赖于 libomp.so，它不会自动加载。如果在 Triton 中使用 PyTorch 后端，您需要在启动 Triton 之前设置 LD_LIBRARY_PATH 以允许根据需要加载 libomp.so。

```
LD_LIBRARY_PATH="$LD_LIBRARY_PATH:/usr/lib/llvm-8/lib"
```

**注意**：在 Jetson 上，必须使用 `--backend-directory` 标志显式指定后端目录。从 23.04 开始，Triton 不再支持 TensorFlow 1.x。如果您想在 23.04 之前的版本中使用 TensorFlow 1.x，需要版本字符串才能使用 TensorFlow 1.x。

```
tritonserver --model-repository=/path/to/model_repo --backend-directory=/path/to/tritonserver/backends \
             --backend-config=tensorflow,version=2
```

**注意**：
[perf_analyzer](https://github.com/triton-inference-server/perf_analyzer/blob/main/README.md)
在 Jetson 上受支持，而 [model_analyzer](model_analyzer_cn.md) 目前不适用于 Jetson。要执行 C API 的 `perf_analyzer`，请使用 CLI 标志 `--service-kind=triton_c_api`：

```shell
perf_analyzer -m graphdef_int32_int32_int32 --service-kind=triton_c_api \
    --triton-server-directory=/opt/tritonserver \
    --model-repository=/workspace/qa/L0_perf_analyzer_capi/models
```

请参考这些[示例](../examples/jetson/README.md)，它们演示了如何在 Jetson 上使用 Triton 推理服务器。