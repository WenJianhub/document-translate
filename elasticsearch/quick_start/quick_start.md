# ElasticsearchClient 8.8.0 快速开始

 

## 使用 Java API Client

​	以下部分提供了有关 Elasticsearch 最常用和一些不太明显的功能教程。如需要完整仓考，请参阅 Elasticsearch 文档，尤其是 REST API 部分。Java API Client 使用 Java API 约定严格遵循此处描述的 JSON 结构。如果您是 Elasticsearch 新手，请务必阅读 Elasticsearch 的快速入门，其中提供了很好的介绍。

* [索引单个文档](#single_document)
* [索引多个文档:rocket:](#multiple_document)
* [通过ID读取文档](#by_id)
* [搜索文件](#searching_for_documents)
* [聚合](#aggregations)

###  <span id="single_document">索引单个文档</span>

Java API Client 提供了几种索引数据的方法：您可以提供自动映射到 JSON 的应用程序对象，或者您可以提供原始数 JSON 数据。使用应用程序对象更适合具有定义良好的域模型的应用程序，而原始 JSON 更适合记录具有半结构化数据的用例。

在下面示例中，我们使用具有sku、名称和价格属性的产品域对象。

> :notebook: 有关索引请求的完整说明，请参阅 [Elasticsearch API 文档]()

#### 使用流畅的DSL

构建请求最直接的方式是使用流畅的DSL。在下面的示例中，我们在产品索引中索引[^产品描述],使用产品的[^SKU]作为索引中的文档标识符。产品对象将使用 Elasticsearch客户端上配置的对象映射到 JSON。

```java
Product product = new Product("sku", "说明", 123.0);

IndexResponse response = esClient.index(i -> i
    .index("products")
    .id(product.getSku())
    .document(product)
);

logger.info("Indexed with version " + response.version());
```

还可以将使用 DSL 创建的对象分配给变量。[^Java API Client]类为此有一个静态的[^of()]方法,

它使用 DSL 语法创建一个对象。

```java
Product product = new Product("sku", "说明", 123.0);

IndexRequest<Product> request = IndexRequest.of(i -> i
    .index("products")
    .id(product.getSku())
    .document(product)
);

IndexResponse response = esClient.index(request);

logger.info("Indexed with version " + response.version());
```

#### 使用经典构造器

如果您更习惯经典的构建器模式，也开始使用它。Builder 对象由流畅的 DSL 语法在底层使用。

```java
Product product = new Product("bk-1", "City bike", 123.0);

IndexRequest.Builder<Product> indexReqBuilder = new IndexRequest.Builder<>();
indexReqBuilder.index("product");
indexReqBuilder.id(product.getSku());
indexReqBuilder.document(product);

IndexResponse response = esClient.index(indexReqBuilder.build());

logger.info("Indexed with version " + response.version());
```

#### 使用异步客户端

上面的示例使用了同步 Elasticsearch 客户端。所有 Elasticsearch API 也可以在异步客户端中使用，使用相同的请求和响应类型。有关更多详细信息，请参阅[阻塞客户端]()和[异步客户端]()。

```java
ElasticsearchAsyncClient esAsyncClient = new ElasticsearchAsyncClient(transport);

Product product = new Product("bk-1", "City bike", 123.0);

esAsyncClient.index(i -> i
    .index("products")
    .id(product.getSku())
    .document(product)
).whenComplete((response, exception) -> {
    if (exception != null) {
        logger.error("Failed to index", exception);
    } else {
        logger.info("Indexed with version " + response.version());
    }
});
```

#### 使用原始 JSON 数据

当要求索引的数据来自外部来源时，必须创建域对象可能很麻烦，或者对于半结构化数据来说是完全不可能的。

可以使用 withJson() 索引任意来源的数据。使用此方法将读取源并将其用于索引请求的文档属性。有关更多的详细信息，请参阅[从JSON 数据创建 API 对象]() 。

```java
Reader input = new StringReader(
    "{'@timestamp': '2022-04-08T13:55:32Z', 'level': 'warn', 'message': 'Some log message'}"
    .replace('\'', '"'));

IndexRequest<JsonData> request = IndexRequest.of(i -> i
    .index("logs")
    .withJson(input)
);

IndexResponse response = esClient.index(request);

logger.info("Indexed with version " + response.version());
```

上述示例的源代码可以在 [Java API Client Tests]() 中找到。



##  <span id="multiple_document">批量：索引多个文档编辑</span>

批量请求允许在一个请求中箱 Elasticsearch 发送多个文档相关的操作。当有多个文档要摄取时，这比使用单独的请求发送每个文档更有效。批量请求可以包含多种操作：

* 创建一个文档，在确保它不存在后对其进行索引，
* 索引文档，在需要时候创创建它，如果存在则替换它，
* 使用脚本或部分文档更新已存在的文档，
* 删除文档。

> :notebook: 有关批量请求的完整说明，请参阅 [Elasticsearch API 文档]()。

### 索引应用对象编辑

BulkRequest 包含一组操作，每个操作都是一个具有多个变体的类型。要创建此请求，主要请求使用构建器对象，每个操作使用流畅的 DSL 会很方便。

下面的示例显示了如何索引列表或应用程序对象。

```java
List<Product> products = fetchProducts();

BulkRequest.Builder br = new BulkRequest.Builder();

for (Product product : products) {
    br.operations(op -> op           
        .index(idx -> idx            
            .index("products")       
            .id(product.getSku())
            .document(product)
        )
    );
}

BulkResponse result = esClient.bulk(br.build());

// Log errors, if any
if (result.errors()) {
    logger.error("Bulk had errors");
    for (BulkResponseItem item: result.items()) {
        if (item.error() != null) {
            logger.error(item.error().reason());
        }
    }
}
```

1. 添加一个操作（记住[列表属性是可加的]()）。 [^op ]是 [^BulkOperation ]的构建器，它是一种变体类型。此类型具有[^index]、[^create]、[^update]、[^delete]变体

2. 选择[^索引]操作变体，[^idx ]是 [^IndexOperation] 的构建器。

3. 设置索引操作的属性，类似于[单文档索引]()：索引名称、标识符和文档

### 索引原始 JSON 数据编辑

批量索引请求的[^document]属性可以是任何可以使用 Elasticsearch 客户端的 JSON 映射器序列化为 JSON 的对象。但是，批量摄取的数据通常以 JSON 文本形式提供（例如磁盘上的文件），解析此 JSON 只是为了重新序列化它以发送批量请求将浪费资源。因此批量操作中的文档也可以是 [^BinaryData] 类型，它们被逐字（不解析）发送到 Elasticsearch 服务器。 

在下面的示例中，我们将使用 Java API 客户端的 [^BinaryData ]从日志目录中读取 json 文件并在批量请求中发送它们。

```java
// List json log files in the log directory
File[] logFiles = logDir.listFiles(
    file -> file.getName().matches("log-.*\\.json")
);

BulkRequest.Builder br = new BulkRequest.Builder();

for (File file: logFiles) {
    FileInputStream input = new FileInputStream(file);
    BinaryData data = BinaryData.of(IOUtils.toByteArray(input), ContentType.APPLICATION_JSON);

    br.operations(op -> op
        .index(idx -> idx
            .index("logs")
            .document(data)
        )
    );
}
```

### 使用 Bulk Ingester 

进行流式摄取编辑 [^BulkIngester] 通过提供一个实用程序类简化了批量 API 的使用，该实用程序类允许在批量请求中透明地对索引/更新/删除操作进行分组。您只需要对 ingester 进行 [^add() ]批量操作，它将负责根据其配置对它们进行分组和批量发送。

当满足以下条件之一时，ingester 将发送批量请求：

* 操作次数超过最大值（默认为 1000）

* 以字节为单位的批量请求大小超过最大值（默认为 5 MiB）

* 自上次请求过期以来的延迟（定期刷新，无默认值）

此外，您可以定义等待 Elasticsearch 执行的并发请求的最大数量（默认为 1）。当达到最大值并且已收集到最大操作数时，向索引器添加新操作将阻塞。这是为了避免通过对客户端应用程序施加背压来使 Elasticsearch 服务器过载。

```java
BulkIngester<Void> ingester = BulkIngester.of(b -> b
    .client(esClient)    
    .maxOperations(100)  
    .flushInterval(1, TimeUnit.SECONDS) 
);

for (File file: logFiles) {
    FileInputStream input = new FileInputStream(file);
    BinaryData data = BinaryData.of(IOUtils.toByteArray(input), ContentType.APPLICATION_JSON);

    ingester.add(op -> op 
        .index(idx -> idx
            .index("logs")
            .document(data)
        )
    );
}

ingester.close(); 
```

