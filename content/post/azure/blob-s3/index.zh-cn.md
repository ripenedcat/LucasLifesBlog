+++
author = "Lucas Huang"
date = '2026-02-04T21:00:00+08:00'
title = "让 Azure Blob Storage 兼容 S3 API"
categories = [
    "Azure"
]
tags = [
    "Azure Blob Storage",
    "S3",
    "S3Proxy"
]
image = "cover.png"

+++

## S3 与 Azure Blob 的关系与差异

Amazon S3 (Simple Storage Service) 和 Azure Blob Storage 都是业界领先的云对象存储服务。它们在核心概念上非常相似：都提供海量、高可用、低成本的非结构化数据存储，都使用扁平的结构（Bucket/Container -> Object/Blob）来组织数据。

然而，在 API 层面，两者存在显著差异。S3 API 已经成为了对象存储领域事实上的标准协议。许多开源工具、数据处理框架（如 Spark, Hadoop）以及商业软件都原生支持 S3 API，或者优先实现了 S3 连接器。

当你希望在 Azure 环境中使用这些仅支持 S3 API 的工具，或者希望将现有的基于 S3 的应用无缝迁移到 Azure Blob Storage 时，协议的不兼容就成了一道障碍。你可能不想重写代码去适配 Azure SDK，也不希望因为存储后端的改变而更换整套工具链。

这时，开源工具 [S3Proxy](https://github.com/gaul/s3proxy) 就派上了用场。它充当了一个中间适配层，对外暴露标准的 S3 API 接口，后端则通过 jclouds 库与 Azure Blob Storage 进行通信。对应用程序而言，它就像在访问一个普通的 S3 服务，而实际上数据是存储在 Azure Blob 上的。

## S3Proxy 配置核心：环境变量

S3Proxy 的行为主要通过环境变量进行配置。要让它连接到 Azure Blob Storage，你需要重点关注以下几个参数：

*   **S3PROXY_AUTHORIZATION**: 设置为 `none` 可以允许匿名访问（仅用于测试），或者配置相应的认证机制。
*   **JCLOUDS_PROVIDER**: 指定后端存储提供商，对于 Azure Blob，这里填写 `azureblob-sdk`,可以兼容Key和Managed Identity来做验证。
*   **JCLOUDS_IDENTITY**: 对应你的 **Azure Storage Account Name**（存储账户名称）。
*   **JCLOUDS_CREDENTIAL**: 对应你的 **Azure Storage Account Key**（存储账户密钥）。
*   **JCLOUDS_ENDPOINT**: Azure Blob 的服务端点，通常格式为 `https://<storageaccountname>.blob.core.windows.net`。
*   **JCLOUDS_AZUREBLOB_AUTH**: 设置认证模式，通常使用 `azureKey`。
*   **S3PROXY_ENDPOINT**: S3Proxy 监听的地址和端口，例如 `http://0.0.0.0:8080`。

### 获取 Azure 凭据

Azure Storage Key 可以通过 Azure Portal 获取，或者如果你安装了 Azure CLI，可以使用以下命令快速获取：

```bash
az storage account keys list -n <storageaccountname> --query '[0].value' -o tsv
```

## 快速运行：使用 Docker

最简单的部署方式是使用 Docker。以下命令将启动一个 S3Proxy 容器，监听本地 8080 端口，并代理到你的 Azure Blob Storage。

请替换命令中的 `<your_storage_account_name>` 和 `<your_storage_account_key>` 为实际值。

```bash
docker run -d --restart=always --name s3proxy \
 -p 8080:8080 \
 -e S3PROXY_AUTHORIZATION=none \
 -e JCLOUDS_IDENTITY="<your_storage_account_name>" \
 -e JCLOUDS_CREDENTIAL="<your_storage_account_key>" \
 -e JCLOUDS_ENDPOINT="https://<your_storage_account_name>.blob.core.windows.net" \
 -e JCLOUDS_PROVIDER=azureblob-sdk \
 -e JCLOUDS_AZUREBLOB_AUTH=azureKey \
 -e S3PROXY_ENDPOINT=http://0.0.0.0:8080 \
 andrewgaul/s3proxy
```

## 验证：像使用 S3 一样使用 Azure Blob

容器启动后，我们可以使用标准的 AWS CLI 工具来验证连接。关键在于使用 `--endpoint-url` 参数指向本地运行的 S3Proxy。

### 1. 列出存储容器 (List Buckets)

此命令将列出 Azure Storage Account 中的所有容器。在 S3 的语义中，Azure 的 Container 等同于 S3 的 Bucket。

```bash
aws --endpoint-url="http://localhost:8080" s3 ls --recursive
```

### 2. 上传文件 (Put Object)

创建一个测试文件并上传到 Azure Blob（假设你已经在 Azure 上创建了一个名为 `test-container` 的容器）：

```bash
echo "Hello Azure Blob via S3 API" > temp.txt
aws --endpoint-url="http://localhost:8080" s3 cp temp.txt s3://test-container/temp.txt
```

输出示例：
```text
upload: ./temp.txt to s3://test-container/temp.txt
```

### 3. 列出文件 (List Objects)

查看 Azure 容器中的文件：

```bash
aws --endpoint-url="http://localhost:8080" s3 ls s3://test-container --recursive
```

你会看到刚才上传的文件。如果你登录 Azure Portal 查看该容器，也会发现文件已经存在。

通过这种方式，你成功打通了 S3 API 到 Azure Blob Storage 的路径，无需修改一行业务代码即可实现跨云存储的兼容。
