Prefect的设置默认存在名为**config.toml**的配置文件中。通常，不应直接编辑此文件以修改Prefect的设置。相反，应该将环境变量用于临时设置，或者为永久设置创建[用户配置文件](https://docs.prefect.io/core/concepts/configuration.html#user-configuration)。

首次导入Prefect时，将对配置文件进行解析，并且可以在**prefect.config**中将其作为活动对象使用。 要访问任何值，请使用点号（例如，prefect.config.tasks.defaults.checkpoint）。

## 环境变量

可以通过环境变量设置任何小写的Prefect变量名的值。为此，请在变量前加上PREFECT__并使用两个下划线（__）分隔键的每个部分。

例如，如果设置PREFECT__TASKS__DEFAULTS__MAX_RETRIES=4，则prefect.config.tasks.defaults.max_retries==4。

> 
> 插入小写键变量
> 
> 环境变量始终被Python代码当作小写配置键插入到运行上下文中，除非它们被指定为本地**Secret**。例如：
> 
> ````bash
> export PREFECT__LOGGING__LEVEL="INFO"
> export PREFECT__CONTEXT__SECRETS__my_KEY="val"
> ````
> 
> 将导致两个配置设置：一个用于config.logging.level，一个用于config.context.secrets.my_KEY（注意后者的大小写可能保留，遵循Prefect Secret的处理规则）。
> 

### 类型自动转换

Prefect将尽力检测环境变量的类型并将其正确的转换。

 - “true”（无论哪个字母大小写）被转换为True
 - “false”（无论哪个字母大小写）被转换为False
 - 解析为整数的字符串将转换为整数
 - 解析为浮点数的字符串将转换为浮点数
 - 所有其他值仍为字符串

## 用户自定义配置

除了环境变量，用户还可以提供自定义配置文件。自定义配置中的所有值都将被加载覆盖默认值，这意味着用户配置仅需要包含要更改的值。

默认情况下，Prefect将在**$HOME/.prefect/config.toml**中查找用户配置文件，但是可以通过设置环境变量PREFECT__USER_CONFIG_PATH来更改该位置。注意变量名称中的双下划线（__），这样可以确保它作为prefect.config.user_config_path在运行时可用。

### 配置优先级

环境变量配置优先级高于用户配置优先级，用户配置优先级高于默认配置优先级。

## TOML

Prefect的配置使用[TOML](https://github.com/toml-lang/toml)编写，TOML是一种结构化文档，支持类型值和嵌套。

### 扩展TOML

Prefect通过两种形式的变量插值扩展了标准TOML。

#### 环境变量插值

当Prefect加载配置时，任何包含以“$”为前缀的环境变量名称的变量标记都将被该环境变量的值替换。例如，如果您的环境中的**DIR=/foo**，则可以在配置中输入以下：

````bash
path = "$DIR/file.txt"
````

在这个场景, 代码中加载使用**prefect.config.path == "/foo/file.txt"**，环境变量在运行上下文中始终被当作小写键来检索数据。

#### 配置文件插入值（自解释）

如果字符串值引用任何其他配置键（用“${”和“}”括起来），则将在运行时内插入该键的值。重复此过程几次以解决多个引用问题。

例如，这可用于从已经定义的变量构建新的复合变量：

````bash
[api]

host = "localhost"
port = "5432"
url = "https://${api.host}:${api.port}"
````

使用

````Python
assert prefect.config.api.url == "https://localhost:5432"
````

甚至基于一个变量的值创建复杂的切换逻辑：

user = "${environments.${environment}}"

````bash
[environments]

    [environments.dev]
        user = "test"

    [environments.prod]
        user = "admin"
````

使用

````Python
assert prefect.config.environment == "prod"
assert prefect.config.user == "admin"
````

#### 变量名可用性验证

首次加载配置时会递归验证。针对无效的配置定义引发**ValueErrors**。检查包括无效变量；因为Config对象具有类似于字典的方法，所以如果它们的任何自定义变量名覆盖了的方法之一，可能会产生问题。例如，“keys”是保留字，因为**Config.keys()**是一个重要的方法。

***

- [Prefect官网](https://www.prefect.io/)
- [英版原文](https://docs.prefect.io/core/concepts/configuration.html)
- [联系译者](https://github.com/listen-lavender)
