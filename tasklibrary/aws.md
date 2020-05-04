与AWS资源进行交互的task集合。

注意所有task都需要一个称为“AWS_CREDENTIALS”的Prefect密钥，该密钥应为带有两个键“ACCESS_KEY”和“SECRET_ACCESS_KEY”的JSON文档。

## LambdaCreate

创建一个Lambda函数的task。

[API参考文档](https://docs.prefect.io/api/latest/tasks/aws.html#lambdacreate)

## LambdaDelete

删除一个Lambda函数的task。

[API参考文档](https://docs.prefect.io/api/latest/tasks/aws.html#lambdadelete)

## LambdaInvoke

唤醒一个Lambda函数的task。

[API参考文档](https://docs.prefect.io/api/latest/tasks/aws.html#lambdainvoke)

## LambdaList

列举Lambda函数的task。

[API参考文档](https://docs.prefect.io/api/latest/tasks/aws.html#lambdalist)

## S3Download

从S3存储bucket下载数据并将其作为字符串返回的task。注意可以选择在运行时提供或覆盖所有初始化参数。

[API参考文档](https://docs.prefect.io/api/latest/tasks/aws.html#s3download)

## S3Upload

用于将字符串数据（例如JSON字符串）上传到S3存储桶的task。注意可以选择在运行时提供或覆盖所有初始化参数。

[API参考文档](https://docs.prefect.io/api/latest/tasks/aws.html#s3upload)

## StepActivate

触发执行AWS Step Function工作流的task。

[API参考文档](https://docs.prefect.io/api/latest/tasks/aws.html#stepactivate)

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/task_library/aws.html)
- [联系译者](https://github.com/listen-lavender)
