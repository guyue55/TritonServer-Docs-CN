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

# 使用 NVIDIA Triton 推理服务器 GKE 市场应用进行基准测试

**目录**
- [模型](#模型)
- [性能](#性能)

## 模型

首先，我们收集一组 TensorFlow 和 TensorRT 模型进行比较：

- 从 Huggingface 获取 [使用 Squad Q&A 任务微调的 Distill Bert](https://huggingface.co/distilbert-base-cased-distilled-squad/tree/main)。`wget https://huggingface.co/distilbert-base-cased-distilled-squad/blob/main/saved_model.tar.gz`
- 从 Huggingface 获取 [使用 Squad Q&A 任务微调的 Bert base](https://huggingface.co/deepset/bert-base-cased-squad2/tree/main)。`wget https://huggingface.co/deepset/bert-base-cased-squad2/blob/main/saved_model.tar.gz`
- 按照 [TensorRT Demo Bert](https://github.com/NVIDIA/TensorRT/tree/master/demo/BERT) 将 BERT base 模型转换为 TensorRT 引擎，选择序列长度为 384 以匹配之前的 2 个 TensorFlow 模型。最后一步，我们选择创建具有 2 个优化配置文件的 TensorRT 引擎，配置文件 0 用于批量大小 1，配置文件 1 用于批量大小 4，运行：`python3 builder.py -m models/fine-tuned/bert_tf_ckpt_base_qa_squad2_amp_384_v19.03.1/model.ckpt -o engines/model.plan -b 8 -s 384 --fp16 --int8 --strict -c models/fine-tuned/bert_tf_ckpt_base_qa_squad2_amp_384_v19.03.1 --squad-json ./squad/train-v2.0.json -v models/fine-tuned/bert_tf_ckpt_base_qa_squad2_amp_384_v19.03.1/vocab.txt --calib-num 100 -iln -imh`。这需要在各自的推理 GPU 上运行（使用 A100 优化的引擎不能用于 T4 上的推理）。

我们将模型放入具有以下结构的 GCS 中，提供了 `config.pbtxt`：
```
    ├── bert_base_trt_gpu
    │   ├── 1
    │   │   └── model.plan
    │   └── config.pbtxt
    ├── bert_base_trt_gpu_seqlen128
    │   ├── 1
    │   │   └── model.plan
    │   └── config.pbtxt
    ├── bert_base_tf_gpu
    │   ├── 1
    │   │   └── model.savedmodel
    │   └── config.pbtxt
    ├── bert_base_tf_cpu
    │   ├── 1
    │   │   └── model.savedmodel
    │   └── config.pbtxt
    ├── bert_distill_tf_gpu
    │   ├── 1
    │   │   └── model.savedmodel
    │   └── config.pbtxt
    └── bert_distill_tf_cpu
        ├── 1
        │   └── model.savedmodel
        └── config.pbtxt
```

部署 Triton GKE 应用程序时，将模型仓库指向包含上述结构和实际模型的目录。

## 性能

我们使用 Triton 的性能分析器来对每个模型进行基准测试，性能分析器位于 GKE 集群的另一个 pod 中。
```bash
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
bash perf_query.sh 35.194.5.119:80 bert_base_trt_gpu 384
```

我们在 n1-standard-96 上部署 CPU BERT BASE 和 Distill BERT，在（n1-standard-4 + T4）上部署 GPU BERT 模型，BERT 模型的序列长度为 384 个标记，并使用 Triton 的性能分析器进行并发扫描来测量延迟/吞吐量。延迟包括 Istio ingress/负载均衡，反映了在同一 GCP 区域中的真实往返成本。

对于所有序列长度为 384 的模型：
CPU BERT BASE：延迟：700ms，吞吐量：12 qps
CPU Distill BERT：延迟：369ms，吞吐量：24 qps

GPU BERT BASE：延迟：230ms，吞吐量：34.7 qps
GPU Distill BERT：延迟：118ms，吞吐量：73.3 qps
GPU TensorRT BERT BASE：延迟：50ms，吞吐量：465 qps

n1-standard-96 的价格为 4.56 美元/小时，n1-standard-4 为 0.19 美元/小时，T4 为 0.35 美元/小时，总计 0.54 美元/小时。在实现更低延迟的同时，使用 T4 上的 TensorRT 进行 BERT 推理的总拥有成本是在 n1-standard-96 上进行 Distill BERT 推理的 163 倍以上。