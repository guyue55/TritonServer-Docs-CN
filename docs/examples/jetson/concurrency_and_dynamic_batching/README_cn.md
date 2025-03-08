<!--
# Copyright (c) 2021-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 并发推理和动态批处理

本示例的目的是演示 Triton Inference Server 的重要特性，如并发模型执行和动态批处理。

我们将使用一个专门构建的可部署人员检测模型，该模型可以从 [Nvidia GPU Cloud (NGC)](https://ngc.nvidia.com/) 下载。

## 获取模型

从 NGC 下载经过剪枝的 [PeopleNet](https://ngc.nvidia.com/catalog/models/nvidia:tlt_peoplenet) 模型。这个模型作为即用型模型提供，您可以使用 `wget` 方法从 NGC 下载：

```shell
wget --content-disposition https://api.ngc.nvidia.com/v2/models/nvidia/tao/peoplenet/versions/pruned_v2.1/zip -O pruned_v2.1.zip
```

或通过 CLI 命令：

```shell
ngc registry model download-version "nvidia/tao/peoplenet:pruned_v2.1"
```

对于后者，您需要设置 [NGC CLI](https://ngc.nvidia.com/setup)。

从 NGC 下载模型后，将压缩文件 `peoplenet_pruned_v2.1.zip` 解压到 `concurrency_and_dynamic_batching/tao/models/peoplenet`。

如果您在 `concurrency_and_dynamic_batching` 目录中有 zip 压缩文件，以下命令将自动将模型放置到正确的位置：

```shell
unzip pruned_v2.1.zip -d $(pwd)/tao/models/peoplenet
```

验证您可以在以下路径看到模型文件 `resnet34_peoplenet_pruned.etlt`：

```
concurrency_and_dynamic_batching
└── tao
       └── models
           └── peoplenet
               ├── labels.txt
               └── resnet34_peoplenet_pruned.etlt
```

## 将模型转换为 TensorRT

获取 `.etlt` 格式的模型文件后，您需要将模型转换为 [TensorRT](https://developer.nvidia.com/tensorrt) 格式。NVIDIA TensorRT 是一个用于高性能深度学习推理的 SDK。它包括深度学习推理优化器和运行时，为深度学习推理应用程序提供低延迟和高吞吐量。最新版本的 JetPack 包含 TensorRT。

要将 `.etlt` 模型转换为 TensorRT 格式，您需要使用 `tao-converter` 工具。

`tao-converter` 工具作为编译后的发布文件提供给不同平台。您可以在 [TLT Getting Started 资源](https://developer.nvidia.com/tlt-get-started)中找到对应您部署系统的下载链接。

下载 `tao-converter` 后，您可能需要在工具所在目录中执行：

```shell
chmod 777 tao-converter
```

我们提供了转换脚本 `tao/convert_peoplenet.sh`，该脚本期望模型位于以下位置：

```shell
tao
└──  models
   └── peoplenet
```

要执行它，您可以将 `tao-converter` 可执行文件放在项目的 `tao` 目录中，并在同一目录中运行：

```shell
bash convert_peoplenet.sh
```

执行后，验证 `model.plan` 文件是否已放置在 `/trtis_model_repo_sample_1/peoplenet/1` 和 `/trtis_model_repo_sample_2/peoplenet/1` 目录中。请注意，我们有两个略有不同的同一模型仓库，用于演示 Triton 的不同特性。

还要注意，此步骤必须在目标硬件上执行：如果您计划在 Jetson 上执行此应用程序，则必须在 Jetson 上进行转换。

要了解更多关于 `tao-converter` 参数的信息，运行：

```shell
./tao-converter -h
```

## 构建应用程序

要编译示例，请拉取以下仓库：
* [https://github.com/triton-inference-server/server](https://github.com/triton-inference-server/server)
* [https://github.com/triton-inference-server/core](https://github.com/triton-inference-server/core)

确保您将下载的发布内容复制到 `$HOME`：

```shell
sudo cp -rf tritonserver2.x.y-jetpack4.6 $HOME/tritonserver
```

在 `concurrency_and_dynamic_batching` 中打开终端并执行以下命令构建应用程序：

```shell
make
```

为 Jetson 提供了示例 Makefile。

## 演示案例 1：并发模型执行

使用 Triton Inference Server，多个模型（或同一模型的多个实例）可以在同一 GPU 或多个 GPU 上同时运行。在此示例中，我们演示如何在单个 Jetson GPU 上运行同一模型的多个实例。

### 运行示例

要从终端执行，在 `concurrency_and_dynamic_batching` 目录中运行：

```shell
LD_LIBRARY_PATH=$HOME/tritonserver/lib ./people_detection -m system -v -r $(pwd)/trtis_model_repo_sample_1 -t 6 -s false -p $HOME/tritonserver
```

参数 `-t` 控制我们要执行的并发推理调用的数量。我们将在同一个样本图像上执行相同的模型，目的是演示设置不同的并发选项如何影响性能。

您可以启用在项目目录中保存检测到的边界框，形式为每个执行线程的原始图像叠加。您可以通过在执行时将参数 `-s` 设置为 `true` 来打开可视化（默认情况下 `-s` 设置为 `false`）。

### 预期输出

执行后，在终端日志中，您将看到以 json 格式显示的 _Model 'peoplenet' Stats_，反映了推理性能。我们还输出 _TOTAL INFERENCE TIME_，它简单地反映了运行应用程序所需的经过时间，包括数据加载、预处理和后处理。

日志中 _Model 'peoplenet' Stats_ 的典型输出如下所示：

```json
{
   "model_stats":[
      {
         "name":"peoplenet",
         "version":"1",
         "last_inference":1626448309997,
         "inference_count":6,
         "execution_count":6,
         "inference_stats":{
            "success":{
               "count":6,
               "ns":574589968
            },
            "fail":{
               "count":0,
               "ns":0
            },
            "queue":{
               "count":6,
               "ns":234669630
            },
            "compute_input":{
               "count":6,
               "ns":194884512
            },
            "compute_infer":{
               "count":6,
               "ns":97322636
            },
            "compute_output":{
               "count":6,
               "ns":47700806
            }
         },
         "batch_stats":[
            {
               "batch_size":1,
               "compute_input":{
                  "count":6,
                  "ns":194884512
               },
               "compute_infer":{
                  "count":6,
                  "ns":97322636
               },
               "compute_output":{
                  "count":6,
                  "ns":47700806
               }
            }
         ]
      }
   ]
}

"TOTAL INFERENCE TIME: 174ms"
```

要了解不同统计信息，请查看[文档](https://github.com/triton-inference-server/server/blob/main/docs/protocol/extension_statistics.md#statistics-extension)。

要查看设置不同的并发值如何影响总执行时间及其在模型统计中反映的组件，您需要修改模型配置文件中的单个参数。

要为模型启用并发模型执行支持，相应的模型配置文件 `trtis_model_repo_sample_1/peoplenet/config.pbtxt` 包含以下内容：

```
instance_group [
  {
    count: 3
    kind: KIND_GPU
  }
]
```

您可以更改同一模型实例允许的推理数量，并观察它如何影响 _Model 'peoplenet' Stats_ 和 _TOTAL INFERENCE TIME_ 中的性能。请注意，在 Jetson 上我们不建议设置太高的值：例如，在 Jetson Xavier AGX 这样的设备上，我们不建议将数值设置大于 6。1-3 范围内的值是最佳的。

在尝试不同的值时，注意它如何影响总推理时间以及一些推理统计信息（如队列和计算时间）。

## 演示案例 2：动态批处理

对于支持批处理的模型，Triton 实现了多种调度和批处理算法，将单个推理请求组合在一起以提高推理吞吐量。在此示例中，我们想要演示启用自动动态批处理如何影响推理性能。

### 运行示例

要观察动态批处理的效果，从 `concurrency_and_dynamic_batching` 目录执行：

```shell
LD_LIBRARY_PATH=$HOME/tritonserver/lib ./people_detection -m system -v -r $(pwd)/trtis_model_repo_sample_2 -t 6 -s false -p $HOME/tritonserver
```

### 预期输出

查看 _Model 'peoplenet' Stats_ 和 _TOTAL INFERENCE TIME_ 以了解动态批处理的效果。可能的结果应该如下所示：

```json
{
   "model_stats":[
      {
         "name":"peoplenet",
         "version":"1",
         "last_inference":1626447787832,
         "inference_count":6,
         "execution_count":2,
         "inference_stats":{
            "success":{
               "count":6,
               "ns":558981051
            },
            "fail":{
               "count":0,
               "ns":0
            },
            "queue":{
               "count":6,
               "ns":49271380
            },
            "compute_input":{
               "count":6,
               "ns":170634044
            },
            "compute_infer":{
               "count":6,
               "ns":338079193
            },
            "compute_output":{
               "count":6,
               "ns":950544
            }
         },
         "batch_stats":[
            {
               "batch_size":1,
               "compute_input":{
                  "count":1,
                  "ns":15955684
               },
               "compute_infer":{
                  "count":1,
                  "ns":29917093
               },
               "compute_output":{
                  "count":1,
                  "ns":152264
               }
            },
            {
               "batch_size":5,
               "compute_input":{
                  "count":1,
                  "ns":30935672
               },
               "compute_infer":{
                  "count":1,
                  "ns":61632420
               },
               "compute_output":{
                  "count":1,
                  "ns":159656
               }
            }
         ]
      }
   ]
}

"TOTAL INFERENCE TIME: 162ms"
```