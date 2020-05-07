与Google Cloud Platform的各个组件交互的task。

## 谷歌云存储

### GCSDownload

用于以字符串形式从Google Cloud Storage下载数据的task模板。

[API参考文档](https://docs.prefect.io/api/latest/tasks/gcp.html#prefect-tasks-google-storage-gcsdownload)

### GCSUpload

用于将数据作为字符串上传到Google Cloud Storage的task模板。

[API参考文档](https://docs.prefect.io/api/latest/tasks/gcp.html#prefect-tasks-google-storage-gcsupload)

### GCSCopy

用于将数据从一个Google Cloud Storage Blob复制到另一个Blob的task模板。

[API参考文档](https://docs.prefect.io/api/latest/tasks/gcp.html#prefect-tasks-google-storage-gcscopy)

## 大数据查询

### BigQueryTask

用于针对Google BigQuery数据表执行查询并（可选的）返回结果的task。

[API参考文档](https://docs.prefect.io/api/latest/tasks/gcp.html#prefect-tasks-google-bigquery-bigquery)

### BigQueryStreamingInsert

通过flow API在Google BigQuery数据表中插入记录的task。

[API参考文档](https://docs.prefect.io/api/latest/tasks/gcp.html#prefect-tasks-google-bigquery-bigquerystreaminginsert)

### CreateBigQueryTable

创建Google BigQuery数据表的task。

[API参考文档](https://docs.prefect.io/api/latest/tasks/gcp.html#prefect-tasks-google-bigquery-createbigquerytable)

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/task_library/gcp.html)
- [联系译者](https://github.com/listen-lavender)

