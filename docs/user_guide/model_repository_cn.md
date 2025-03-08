<!--
# Copyright 2018-2024, NVIDIA CORPORATION & AFFILIATES. All rights reserved.
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

# 模型仓库

**这是您第一次设置模型仓库吗？** 请查看[这些教程](https://github.com/triton-inference-server/tutorials/tree/main/Conceptual_Guide/Part_1-model_deployment#setting-up-the-model-repository)开始您的 Triton 之旅！

Triton 推理服务器从服务器启动时指定的一个或多个模型仓库中提供模型服务。在 Triton 运行期间，可以按照[模型管理](model_management.md)中的描述修改正在服务的模型。

## 仓库布局

这些仓库路径在使用 --model-repository 选项启动 Triton 时指定。可以多次指定 --model-repository 选项以包含来自多个仓库的模型。组成模型仓库的目录和文件必须遵循所需的布局。假设仓库路径按如下方式指定：

```bash
$ tritonserver --model-repository=<model-repository-path>
```

相应的仓库布局必须是：

```
  <model-repository-path>/
    <model-name>/
      [config.pbtxt]
      [<output-labels-file> ...]
      [configs]/
        [<custom-config-file> ...]
      <version>/
        <model-definition-file>
      <version>/
        <model-definition-file>
      ...
    <model-name>/
      [config.pbtxt]
      [<output-labels-file> ...]
      [configs]/
        [<custom-config-file> ...]
      <version>/
        <model-definition-file>
      <version>/
        <model-definition-file>
      ...
    ...
```

在顶级模型仓库目录中必须有零个或多个 <model-name> 子目录。每个 <model-name> 子目录包含相应模型的仓库信息。config.pbtxt 文件描述了模型的[模型配置](model_configuration.md)。对于某些模型，config.pbtxt 是必需的，而对于其他模型则是可选的。有关更多信息，请参阅[自动生成的模型配置](model_configuration.md#auto-generated-model-configuration)。

每个 <model-name> 目录可以包含一个可选的 configs 子目录。在 configs 目录中必须有零个或多个带有 .pbtxt 文件扩展名的 <custom-config-file>。有关 Triton 如何处理自定义模型配置的更多信息，请参阅[自定义模型配置](https://github.com/triton-inference-server/server/blob/main/docs/user_guide/model_configuration.md#custom-model-configuration)。

每个 <model-name> 目录必须至少有一个数字子目录，表示模型的版本。有关 Triton 如何处理模型版本的更多信息，请参阅[模型版本](#model-versions)。每个模型由特定的[后端](https://github.com/triton-inference-server/backend/blob/main/README.md)执行。在每个版本子目录中必须有该后端所需的文件。例如，使用 TensorRT、PyTorch、ONNX、OpenVINO 和 TensorFlow 等框架后端的模型必须提供[特定于框架的模型文件](#model-files)。

## 模型仓库位置

Triton 可以从一个或多个本地可访问的文件路径、Google Cloud Storage、Amazon S3 和 Azure Storage 访问模型。

### 本地文件系统

对于本地可访问的文件系统，必须指定绝对路径。

```bash
$ tritonserver --model-repository=/path/to/model/repository ...
```

### 使用环境变量的云存储

#### Google Cloud Storage

对于位于 Google Cloud Storage 中的模型仓库，仓库路径必须以 gs:// 为前缀。

```bash
$ tritonserver --model-repository=gs://bucket/path/to/model/repository ...
```

使用 Google Cloud Storage 时，凭证按以下顺序获取和尝试：
1. [GOOGLE_APPLICATION_CREDENTIALS 环境变量](https://cloud.google.com/docs/authentication/application-default-credentials#GAC)
   - 环境变量应设置并包含凭证 JSON 文件的位置。
   - 首先尝试授权用户凭证，然后是服务账号凭证。
2. [附加的服务账号](https://cloud.google.com/docs/authentication/application-default-credentials#attached-sa)
   - 应该能够获取 [Authorization HTTP 头](https://googleapis.dev/cpp/google-cloud-storage/1.42.0/classgoogle_1_1cloud_1_1storage_1_1oauth2_1_1ComputeEngineCredentials.html#a8c3a5d405366523e2f4df06554f0a676) 的值。
3. 匿名凭证（也称为公共存储桶）
   - 存储桶（和对象）应该授予所有用户 `get` 和 `list` 权限。
   - 授予此类权限的一种方法是为存储桶的 "allUsers" 添加 [storage.objectViewer](https://cloud.google.com/storage/docs/access-control/iam-roles#standard-roles) 和 [storage.legacyBucketReader](https://cloud.google.com/storage/docs/access-control/iam-roles#legacy-roles) 预定义角色，例如：
        ```
        $ gsutil iam ch allUsers:objectViewer "${BUCKET_URL}"
        $ gsutil iam ch allUsers:legacyBucketReader "${BUCKET_URL}"
        ```

默认情况下，Triton 会在临时文件夹中创建远程模型仓库的本地副本，该文件夹在 Triton 服务器关闭后会被删除。如果您想控制远程模型仓库复制到的位置，可以将 `TRITON_GCS_MOUNT_DIRECTORY` 环境变量设置为指向本地机器上现有文件夹的路径。

```bash
export TRITON_GCS_MOUNT_DIRECTORY=/path/to/your/local/directory
```

**确保 `TRITON_GCS_MOUNT_DIRECTORY` 在您的本地机器上存在且为空。**

#### S3

对于位于 Amazon S3 中的模型仓库，路径必须以 s3:// 为前缀。

```bash
$ tritonserver --model-repository=s3://bucket/path/to/model/repository ...
```

对于 S3 的本地或私有实例，前缀 s3:// 后必须跟随主机和端口（用分号分隔），然后是存储桶路径。

```bash
$ tritonserver --model-repository=s3://host:port/bucket/path/to/model/repository ...
```

默认情况下，Triton 使用 HTTP 与您的 S3 实例通信。如果您的 S3 实例支持 HTTPS，并且您希望 Triton 使用 HTTPS 协议与之通信，可以在模型仓库路径中通过在主机名前加上 https:// 来指定。

```bash
$ tritonserver --model-repository=s3://https://host:port/bucket/path/to/model/repository ...
```

使用 S3 时，可以通过使用 [aws config](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html) 命令或通过相应的[环境变量](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html)来传递凭证和默认区域。如果设置了环境变量，它们将具有更高的优先级，Triton 将使用它们而不是使用 aws config 命令设置的凭证。

默认情况下，Triton 会在临时文件夹中创建远程模型仓库的本地副本，该文件夹在 Triton 服务器关闭后会被删除。如果您想控制远程模型仓库复制到的位置，可以将 `TRITON_AWS_MOUNT_DIRECTORY` 环境变量设置为指向本地机器上现有文件夹的路径。

```bash
export TRITON_AWS_MOUNT_DIRECTORY=/path/to/your/local/directory
```

**确保 `TRITON_AWS_MOUNT_DIRECTORY` 在您的本地机器上存在且为空。**

#### Azure Storage

对于位于 Azure Storage 中的模型仓库，仓库路径必须以 as:// 为前缀。

```bash
$ tritonserver --model-repository=as://account_name/container_name/path/to/model/repository ...
```

使用 Azure Storage 时，您必须设置 `AZURE_STORAGE_ACCOUNT` 和 `AZURE_STORAGE_KEY` 环境变量为有权访问 Azure Storage 仓库的账户。

如果您不知道您的 `AZURE_STORAGE_KEY` 并且已正确配置了 Azure CLI，以下是如何找到与您的 `AZURE_STORAGE_ACCOUNT` 对应的密钥的示例：

```bash
$ export AZURE_STORAGE_ACCOUNT="account_name"
$ export AZURE_STORAGE_KEY=$(az storage account keys list -n $AZURE_STORAGE_ACCOUNT --query "[0].value")
```

默认情况下，Triton 会在临时文件夹中创建远程模型仓库的本地副本，该文件夹在 Triton 服务器关闭后会被删除。如果您想控制远程模型仓库复制到的位置，可以将 `TRITON_AZURE_MOUNT_DIRECTORY` 环境变量设置为指向本地机器上现有文件夹的路径。

```bash
export TRITON_AZURE_MOUNT_DIRECTORY=/path/to/your/local/directory
```

**确保 `TRITON_AZURE_MOUNT_DIRECTORY` 在您的本地机器上存在且为空。**