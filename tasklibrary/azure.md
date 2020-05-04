与Azure云进行交互的task集合。

注意，所有task都需要一个称为“AZ_CREDENTIALS”的Prefect秘钥，它应该是带有两个键“ACCOUNT_NAME”和“ACCOUNT_KEY/SAS_TOKEN”的JSON文档。

### BlobStorageDownload

从Blob存储容器下载数据并将其作为字符串返回的task。注意可以选择在运行时提供或覆盖所有初始化参数。

[API参考文档](https://docs.prefect.io/api/latest/tasks/azure.html#prefect-tasks-azure-blobstorage-blobstoragedownload)

### BlobStorageUpload

将字符串数据（例如JSON字符串）上传到Blob存储的task。注意可以选择在运行时提供或覆盖所有初始化参数。

[API参考文档](https://docs.prefect.io/api/latest/tasks/azure.html#prefect-tasks-azure-blobstorage-blobstorageupload)

### CosmosDBCreateItem

在Cosmos数据库中创建项目的task。注意可以选择在运行时提供或覆盖所有初始化参数。

[API参考文档](https://docs.prefect.io/api/latest/tasks/azure.html#prefect-tasks-azure-cosmosdb-cosmosdbcreateitem)

### CosmosDBReadItems

从Azure Cosmos数据库读取项目的task。注意可以选择在运行时提供或覆盖所有初始化参数。

[API参考文档](https://docs.prefect.io/api/latest/tasks/azure.html#prefect-tasks-azure-cosmosdb-cosmosdbreaditems)

### CosmosDBQueryItems

从Azure Cosmos数据库查询项目的task。注意可以选择在运行时提供或覆盖所有初始化参数。

[API参考文档](https://docs.prefect.io/api/latest/tasks/azure.html#prefect-tasks-azure-cosmosdb-cosmosdbqueryitems)

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/task_library/azure.html)
- [联系译者](https://github.com/listen-lavender)
