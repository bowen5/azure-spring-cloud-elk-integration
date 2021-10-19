# Azure Spring Cloud 与 ELK 集成文档

1. 准备 ELK 环境

    如果没有的话，可以参考 [Getting started with the Azure Marketplace](https://www.elastic.co/guide/en/elastic-stack-deploy/current/azure-marketplace-getting-started.html) 在 Azure Marketplace 中创建。

    - 访问 Elasticsearch

      ![image.png](/resources/elasticsearch.jpg)

    - 访问 Kibana

      ![image.png](/resources/kibana.jpg)

2. 安装 Filebeat

    Filebeat 可以被安装在任何能够访问 Elasticsearch 和 Kibana 的机器上。也可以直接安装在 Kibana 节点上。

    2.1 连接至想要安装 Filebeat 的节点

    2.2 下载 Filebeat，注意版本和操作系统需要和当前 ELK 所使用的一致，本文档使用的 ELK 版本是 7.11.1。

    ```
    curl -L -O https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.11.1-linux-x86_64.tar.gz
    tar xzvf filebeat-7.11.1-linux-x86_64.tar.gz
    cd filebeat-7.11.1-linux-x86_64/
    ```

3. 配置 Filebeat 以连接 Elasticsearch 和 Kibana

    修改 `filebeat.yml`，修改如下配置：

    ```
    output.elasticsearch:
      hosts: ["<es_url>"]
      username: "elastic"
      password: "<password>"
    setup.kibana:
      host: "<kibana_url>"
    ```

    `<password>` 是 `elastic` 用户的密码。`<es_url>` 是 Elasticsearch 的 URL，`<kibana_url>` 是 Kibana 的 URL。

    本示例配置如下：

    ```
    output.elasticsearch:
      hosts: ["lb-voxslwvc3cv3a.westus2.cloudapp.azure.com:9200"]
      username: "elastic"
      password: "Test1234!!!!"
    setup.kibana:
      host: "kb-uznkju4ma7hpg.westus2.cloudapp.azure.com:5601"
    ```

4. 启用 Filebeat 的 azure 模块

    ![image.png](/resources/enableazure.jpg)

5. 创建 Azure Event Hub

    参考 [Create an event hub using Azure portal](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-create) 来创建 Azure Event Hub namepace 以及 Event Hub。如下图，创建了一个名为 `petclinic-eventhub` 的 Event Hub namepsace 以及其下名为 `springcloud` 的 Event Hub。

    ![image.png](/resources/eventhubns.jpg)

    ![image.png](/resources/eventhub.jpg)

6. 将 Azure Spring Cloud 日志导入 Event Hub

    配置 Azure Spring Cloud Diagnostic settings，将日志导入第 5 步中创建的 Event Hub。

    ![image.png](/resources/springcloudtoeventhub.jpg)

7. 创建 Azure Storage Account

    Filebeat azure 模块需要使用 Azure Blob Storage 存储信息。参考 [Create a storage account](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-create?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&tabs=azure-portal) 来创建 Azure Storage Account。

    ![image.png](/resources/storageaccount.jpg)

8. 配置 Filebeat azure 模块

    回到 Filebeat 目录，修改 `modules.d/azure.yml` 文件。

    ```
    - module: azure
      activitylogs:
        enabled: false

      platformlogs:
        enabled: true
        var:          
          eventhub: "springcloud"
          consumer_group: "$Default"          
          connection_string: "Endpoint=sb://petclinic-eventhub.servicebus.windows.net/;SharedAccessKeyName=RootManageSharedAccessKey;SharedAccessKey=****"
          storage_account: "springelkstorage"
          storage_account_key: "****"
    ```

    - 关闭 Activity logs，在此集成中不需要。
    - 打开 Platform logs。
    - `eventhub` 填写第 5 步中创建的 Event Hub 名字。注意：不是 Event Hub namespace 名字。
    - `consumer_group` 保留默认值。
    - `connection_string` 填写 Event Hub connection string。可参考 [Get an Event Hubs connection string](https://docs.microsoft.com/en-us/azure/event-hubs/event-hubs-get-connection-string) 获取。
    - `storage_account` 填写第 7 步中创建的 Azure Storage Account 名字。
    - `storage_account_key` 填写第 7 步中创建的 Azure Storage Account Key。可参考 [View account access keys](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-keys-manage?toc=%2Fazure%2Fstorage%2Fblobs%2Ftoc.json&tabs=azure-portal#view-account-access-keys) 获取。
<br></br>

9. 启动 Filebeat

    ```
    ./filebeat setup
    ./filebeat -e
    ```

    ![image.png](/resources/runfilebeat.jpg)


9. 接下来就能在 ELK 里看到来自 Azure Spring Cloud 的日志了。