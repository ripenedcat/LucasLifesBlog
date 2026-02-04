+++
author = "Lucas Huang"
date = '2026-02-04T21:00:00+08:00'
title = "Making Azure Blob Storage Compatible with S3 API"
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

## The Relationship and Differences Between S3 and Azure Blob

Amazon S3 (Simple Storage Service) and Azure Blob Storage are both industry-leading cloud object storage services. They are very similar in core concepts: both provide massive, highly available, low-cost unstructured data storage, and both use a flat structure (Bucket/Container -> Object/Blob) to organize data.

However, there are significant differences at the API level. The S3 API has become the de facto standard protocol in the object storage field. Many open-source tools, data processing frameworks (such as Spark, Hadoop), and commercial software natively support the S3 API or have prioritized S3 connectors.

When you want to use these tools that only support the S3 API in an Azure environment, or wish to seamlessly migrate existing S3-based applications to Azure Blob Storage, protocol incompatibility becomes an obstacle. You might not want to rewrite code to adapt to the Azure SDK, nor do you wish to change your entire toolchain just because the storage backend has changed.

This is where the open-source tool [S3Proxy](https://github.com/gaul/s3proxy) comes in handy. It acts as an intermediate adaptation layer, exposing standard S3 API interfaces externally while communicating with Azure Blob Storage via the jclouds library on the backend. For applications, it feels like accessing a regular S3 service, but the data is actually stored on Azure Blob.

## S3Proxy Configuration Core: Environment Variables

S3Proxy's behavior is primarily configured via environment variables. To connect it to Azure Blob Storage, you need to focus on the following parameters:

*   **S3PROXY_AUTHORIZATION**: Set to `none` to allow anonymous access (for testing only), or configure appropriate authentication mechanisms.
*   **JCLOUDS_PROVIDER**: Specifies the backend storage provider. For Azure Blob, use `azureblob-sdk`, which supports both Key and Managed Identity for authentication.
*   **JCLOUDS_IDENTITY**: Corresponds to your **Azure Storage Account Name**.
*   **JCLOUDS_CREDENTIAL**: Corresponds to your **Azure Storage Account Key**.
*   **JCLOUDS_ENDPOINT**: The service endpoint for Azure Blob, typically in the format `https://<storageaccountname>.blob.core.windows.net`.
*   **JCLOUDS_AZUREBLOB_AUTH**: Sets the authentication mode, usually `azureKey`.
*   **S3PROXY_ENDPOINT**: The address and port S3Proxy listens on, e.g., `http://0.0.0.0:8080`.

### Getting Azure Credentials

The Azure Storage Key can be obtained via the Azure Portal. Alternatively, if you have the Azure CLI installed, you can quickly get it using the following command:

```bash
az storage account keys list -n <storageaccountname> --query '[0].value' -o tsv
```

## Quick Run: Using Docker

The simplest deployment method is using Docker. The following command starts an S3Proxy container, listening on local port 8080, and proxying to your Azure Blob Storage.

Please replace `<your_storage_account_name>` and `<your_storage_account_key>` in the command with your actual values.

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

## Verification: Using Azure Blob Like S3

After the container starts, we can use standard AWS CLI tools to verify the connection. The key is to use the `--endpoint-url` parameter to point to the locally running S3Proxy.

### 1. List Storage Containers (List Buckets)

This command lists all containers in the Azure Storage Account. In S3 semantics, an Azure Container is equivalent to an S3 Bucket.

```bash
aws --endpoint-url="http://localhost:8080" s3 ls --recursive
```

### 2. Upload File (Put Object)

Create a test file and upload it to Azure Blob (assuming you have already created a container named `test-container` on Azure):

```bash
echo "Hello Azure Blob via S3 API" > temp.txt
aws --endpoint-url="http://localhost:8080" s3 cp temp.txt s3://test-container/temp.txt
```

Example Output:
```text
upload: ./temp.txt to s3://test-container/temp.txt
```

### 3. List Files (List Objects)

View files in the Azure container:

```bash
aws --endpoint-url="http://localhost:8080" s3 ls s3://test-container --recursive
```

You will see the file you just uploaded. If you log in to the Azure Portal and check the container, you will find the file there as well.

In this way, you have successfully bridged the path from S3 API to Azure Blob Storage, enabling cross-cloud storage compatibility without modifying a single line of business code.
