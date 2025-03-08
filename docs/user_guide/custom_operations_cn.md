<!--
# Copyright 2019-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 自定义操作

Triton Inference Server 部分支持允许自定义操作的建模框架。自定义操作可以在构建时或启动时添加到 Triton 中，并对所有已加载的模型可用。

## TensorRT

TensorRT 允许用户创建[自定义层](https://docs.nvidia.com/deeplearning/tensorrt/developer-guide/index.html#extending)，然后可以在 TensorRT 模型中使用这些层。要在 Triton 中运行这些模型，必须使自定义层可用。

要使自定义层对 Triton 可用，TensorRT 自定义层实现必须编译成一个或多个共享库，然后使用 LD_PRELOAD 加载到 Triton 中。例如，假设您的 TensorRT 自定义层编译到 libtrtcustom.so 中，使用以下命令启动 Triton 可以使这些自定义层对所有 TensorRT 模型可用。

```bash
$ LD_PRELOAD=libtrtcustom.so:${LD_PRELOAD} tritonserver --model-repository=/tmp/models ...
```

这种方法的一个限制是自定义层必须与模型仓库本身分开管理。更严重的是，如果多个共享库之间存在自定义层名称冲突，目前没有办法处理。

在构建自定义层共享库时，使用与 Triton 中相同版本的 TensorRT 很重要。您可以在 [Triton 发布说明](https://docs.nvidia.com/deeplearning/triton-inference-server/release-notes/index.html)中找到 TensorRT 版本。确保使用正确版本 TensorRT 的简单方法是使用与 Triton 容器对应的 [NGC TensorRT 容器](https://ngc.nvidia.com/catalog/containers/nvidia:tensorrt)。例如，如果您使用的是 24.09 版本的 Triton，请使用 24.09 版本的 TensorRT 容器。

## TensorFlow

TensorFlow 允许用户[添加自定义操作](https://www.tensorflow.org/guide/create_op)，然后可以在 TensorFlow 模型中使用这些操作。您可以通过两种方式将自定义 TensorFlow 操作加载到 Triton 中：
* 在模型加载时，通过在模型配置中列出它们。
* 在服务器启动时，通过使用 LD_PRELOAD。

要通过模型配置注册您的自定义操作库，您可以将其作为附加字段包含进来。请参考以下配置示例。

```bash
$ model_operations { op_library_filename: "path/to/libtfcustom.so" }
```

请注意，即使模型是在运行时加载的，多个模型也可以使用自定义操作。目前没有办法释放自定义操作，所以它们将一直可用，直到 Triton 关闭。

您也可以通过 LD_PRELOAD 注册您的自定义操作库。例如，假设您的 TensorFlow 自定义操作编译到 libtfcustom.so 中，使用以下命令启动 Triton 可以使这些操作对所有 TensorFlow 模型可用。

```bash
$ LD_PRELOAD=libtfcustom.so:${LD_PRELOAD} tritonserver --model-repository=/tmp/models ...
```

使用这种方法，所有 TensorFlow 自定义操作都依赖于一个 TensorFlow 共享库，该库在加载时必须对自定义共享库可用。实际上，这意味着您必须确保在发出上述命令之前，/opt/tritonserver/backends/tensorflow1 或 /opt/tritonserver/backends/tensorflow2 在库路径上。有几种方法可以控制库路径，一种常见的方法是使用 LD_LIBRARY_PATH。您可以在 "docker run" 命令中或在容器内设置 LD_LIBRARY_PATH。

```bash
$ export LD_LIBRARY_PATH=/opt/tritonserver/backends/tensorflow1:$LD_LIBRARY_PATH
```

这种方法的一个限制是自定义操作必须与模型仓库本身分开管理。更严重的是，如果多个共享库之间存在自定义层名称冲突，目前没有办法处理。

在构建自定义操作共享库时，使用与 Triton 中相同版本的 TensorFlow 很重要。您可以在 [Triton 发布说明](https://docs.nvidia.com/deeplearning/triton-inference-server/release-notes/index.html)中找到 TensorFlow 版本。确保使用正确版本 TensorFlow 的简单方法是使用与 Triton 容器对应的 [NGC TensorFlow 容器](https://ngc.nvidia.com/catalog/containers/nvidia:tensorflow)。例如，如果您使用的是 24.09 版本的 Triton，请使用 24.09 版本的 TensorFlow 容器。

## PyTorch

Torchscript 允许用户[添加自定义操作](https://pytorch.org/tutorials/advanced/torch_script_custom_ops.html)，然后可以在 Torchscript 模型中使用这些操作。通过使用 LD_PRELOAD，您可以将自定义 C++ 操作加载到 Triton 中。例如，如果您按照 [pytorch/extension-script](https://github.com/pytorch/extension-script) 仓库中的说明操作，并且您的 Torchscript 自定义操作编译到 libpytcustom.so 中，使用以下命令启动 Triton 可以使这些操作对所有 PyTorch 模型可用。由于所有 PyTorch 自定义操作都依赖于一个或多个 PyTorch 共享库，这些库在加载时必须对自定义共享库可用。实际上，这意味着您必须确保在启动服务器时 /opt/tritonserver/backends/pytorch 在库路径上。有几种方法可以控制库路径，一种常见的方法是使用 LD_LIBRARY_PATH。

```bash
$ LD_LIBRARY_PATH=/opt/tritonserver/backends/pytorch:$LD_LIBRARY_PATH LD_PRELOAD=libpytcustom.so:${LD_PRELOAD} tritonserver --model-repository=/tmp/models ...
```

这种方法的一个限制是自定义操作必须与模型仓库本身分开管理。更严重的是，如果多个共享库之间或用于在 PyTorch 中注册它们的句柄存在自定义层名称冲突，目前没有办法处理。

从 Triton 20.07 版本开始，[TorchVision 操作](https://github.com/pytorch/vision)将包含在 PyTorch 后端中，因此不必将其显式添加为自定义操作。

在构建自定义操作共享库时，使用与 Triton 中相同版本的 PyTorch 很重要。您可以在 [Triton 发布说明](https://docs.nvidia.com/deeplearning/triton-inference-server/release-notes/index.html)中找到 PyTorch 版本。确保使用正确版本 PyTorch 的简单方法是使用与 Triton 容器对应的 [NGC PyTorch 容器](https://ngc.nvidia.com/catalog/containers/nvidia:pytorch)。例如，如果您使用的是 24.09 版本的 Triton，请使用 24.09 版本的 PyTorch 容器。

## ONNX

ONNX Runtime 允许用户[添加自定义操作](https://onnxruntime.ai/docs/reference/operators/add-custom-op.html)，然后可以在 ONNX 模型中使用这些操作。要注册您的自定义操作库，您需要将其作为附加字段包含在模型配置中。例如，如果您按照 [microsoft/onnxruntime](https://github.com/microsoft/onnxruntime) 仓库中的[这个示例](https://github.com/microsoft/onnxruntime/blob/master/onnxruntime/test/shared_lib/test_inference.cc)操作，并且您的 ONNXRuntime 自定义操作编译到 libonnxcustom.so 中，将以下内容添加到您的模型配置中可以使这些操作对该特定 ONNX 模型可用。

```bash
$ model_operations { op_library_filename: "/path/to/libonnxcustom.so" }
```

在构建自定义操作共享库时，使用与 Triton 中相同版本的 ONNXRuntime 很重要。您可以在 [Triton 发布说明](https://docs.nvidia.com/deeplearning/triton-inference-server/release-notes/index.html)中找到 ONNXRuntime 版本。