# 基于Triton Server的模型编排方案

## 1. Triton Server模型编排功能分析

### 1.1 Ensemble模型概述

Triton Server的Ensemble（集成）模型代表了一个由一个或多个模型组成的**流水线**，以及这些模型之间输入和输出张量的连接关系。Ensemble模型旨在封装涉及多个模型的流程，例如"数据预处理 -> 推理 -> 数据后处理