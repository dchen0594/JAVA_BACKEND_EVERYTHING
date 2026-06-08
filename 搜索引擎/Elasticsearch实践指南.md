# Elasticsearch 实践指南（通俗易懂版）

> 本文从零讲解 Elasticsearch（简称 ES）是什么、为什么用、怎么用，涵盖 Docker 部署、Spring Boot 3 集成、常用 API、避坑指南与监控。  
> **目录跳转**：章节使用显式 `id` 锚点，保证 GitHub / VS Code / Cursor 预览可稳定跳转。

---

## 目录

- [1. 为什么要用 Elasticsearch？](#sec-01-why-es)
- [2. ES 适合什么场景？](#sec-02-scenarios)
- [3. ES 核心概念（先搞懂这些再动手）](#sec-03-concepts)
- [4. ES 架构：单机 vs 集群](#sec-04-architecture)
- [5. Docker 部署 ES（开发/测试）](#sec-05-docker)
- [6. 集群部署要点（生产）](#sec-06-cluster)
- [7. Spring Boot 3 集成 ES](#sec-07-springboot3)
- [8. 常用 API 详解](#sec-08-apis)
- [9. 使用中的常见坑与防范](#sec-09-pitfalls)
- [10. 监控工具与使用方式](#sec-10-monitoring)
- [11. 学习路径与速查](#sec-11-cheatsheet)

---

<h2 id="sec-01-why-es">1. 为什么要用 Elasticsearch？</h2>

### 1.1 一句话理解 ES

**Elasticsearch 是一个基于 Lucene 的分布式搜索与分析引擎**，你可以把它想象成：

> 一个「超级快的全文检索数据库 + 轻量分析平台」。

传统 MySQL 擅长精确查询（`WHERE id = 1`），但面对下面这类需求会很吃力：

- 搜「苹果手机」能匹配「iPhone」「Apple 手机」
- 搜「北京 天气」要按相关度排序，不是简单 `LIKE '%北京%'`
- 日志、商品、文章数据量上亿，还要秒级返回
- 需要聚合统计：按城市分组、按时间段看趋势

ES 就是为解决这些问题而生的。

### 1.2 ES 能做什么

| 能力 | 说明 |
|------|------|
| **全文检索** | 分词、模糊匹配、相关度评分 |
| **结构化查询** | term、range、bool 组合，类似 SQL 但更灵活 |
| **聚合分析** | 类似 `GROUP BY` + 统计函数，适合报表 |
| **近实时搜索** | 写入后通常 1 秒内可搜（refresh 机制） |
| **水平扩展** | 数据分片到多台机器，集群扩容 |

### 1.3 为什么不用 MySQL 全文索引？

| 对比 | MySQL 全文索引 | Elasticsearch |
|------|----------------|---------------|
| 分词能力 | 弱（中文尤其弱） | 强（IK 等分词器） |
| 相关度排序 | 基本无 | 核心能力（BM25 等） |
| 大数据量 | 性能下降明显 | 分片分布式，易扩展 |
| 复杂聚合 | 复杂 SQL，性能差 | 原生 agg，适合分析 |
| 运维 | 简单 | 相对复杂，需专门学习 |

**结论**：MySQL 做业务主库，ES 做搜索/分析，是常见架构。

### 1.4 ES 不是什么

- ❌ 不是 MySQL 的替代品（不适合强事务、频繁更新单条记录）
- ❌ 不是消息队列（虽然早期 Logstash 常一起用）
- ❌ 不是实时 OLTP 数据库（更新/delete 有成本，最终一致性）

---

<h2 id="sec-02-scenarios">2. ES 适合什么场景？</h2>

### 2.1 典型业务场景

| 场景 | 举例 |
|------|------|
| **站内搜索** | 电商商品搜索、文章搜索、文档搜索 |
| **日志分析（ELK/EFK）** | 应用日志、Nginx 访问日志集中检索 |
| **监控与 APM** | Metricbeat 指标、异常日志告警 |
| **推荐/标签** | 按用户行为聚合标签、相似内容 |
| **地理搜索** | 附近的人、附近的外卖（geo 查询） |
| **安全分析（SIEM）** | 审计日志关联分析 |

### 2.2 一个电商搜索的例子

用户在搜索框输入「红色 连衣裙 夏季」：

```
用户输入 → ES 分词 → 匹配 title/description/tags
         → 按相关度 + 销量 + 价格权重排序
         → 100ms 内返回结果
```

MySQL `LIKE` 很难做好相关度和性能；ES 是标准解法。

### 2.3 什么时候**不建议**用 ES

- 数据量小（几万条），MySQL + 简单索引够用
- 强一致事务（转账、库存扣减）—— 仍用 MySQL
- 频繁单条 update（ES 是「删了再写」或部分更新，成本高）
- 团队无人运维 ES，且没有监控体系

---

<h2 id="sec-03-concepts">3. ES 核心概念（先搞懂这些再动手）</h2>

### 3.1 与 MySQL 对照（最好记的方式）

| MySQL | Elasticsearch | 说明 |
|-------|---------------|------|
| 数据库 Database | **Index（索引）** | 一类数据的逻辑集合 |
| 表 Table | **Type（类型）** | 7.x 已废弃，现在一索引一类型 `_doc` |
| 行 Row | **Document（文档）** | JSON 对象，最小存储单元 |
| 列 Column | **Field（字段）** | 文档里的 key |
| 索引 Index | **Inverted Index（倒排索引）** | 词 → 文档列表 |
| 表结构 Schema | **Mapping（映射）** | 字段类型、分词器定义 |
| 分区 | **Shard（分片）** | 水平拆分 |
| 主从复制 | **Replica（副本）** | 高可用 + 读扩展 |

### 3.2 集群相关概念

```
Cluster（集群）
  └── Node（节点）× N
        └── Index（索引）
              └── Shard（主分片 Primary）+ Replica（副本分片）
                    └── Segment（段，Lucene 底层文件）
```

| 概念 | 通俗解释 |
|------|----------|
| **Cluster** | 一组 ES 节点的集合，有个集群名（默认 `elasticsearch`） |
| **Node** | 一台 ES 进程；角色有 master、data、ingest、 coordinating |
| **Primary Shard** | 主分片，数据写入这里；创建索引时数量固定，之后难改 |
| **Replica Shard** | 副本，主分片挂了顶上；也可分担读请求 |
| **Document _id** | 文档唯一 ID，可自定义或 ES 自动生成 |

### 3.3 Mapping 与字段类型（常用）

| 类型 | 用途 | 能否全文搜索 |
|------|------|--------------|
| `text` | 长文本，会分词 | ✅ |
| `keyword` | 精确值：状态码、标签、邮箱 | ❌（精确匹配） |
| `long/integer/double` | 数值 | 范围查询 |
| `date` | 日期 | 范围查询 |
| `boolean` | 布尔 | term 查询 |
| `nested` | 嵌套对象数组 | 需 nested 查询 |
| `geo_point` | 经纬度 | geo 距离查询 |

**重要**：`text` 与 `keyword` 常配合使用：

```json
{
  "title": {
    "type": "text",
    "fields": {
      "keyword": { "type": "keyword", "ignore_above": 256 }
    }
  }
}
```

- `title` → 全文搜
- `title.keyword` → 排序、聚合、精确过滤

### 3.4 倒排索引（理解搜索为何快）

正排：文档 ID → 内容  
倒排：词项 → [文档1, 文档5, 文档99]

搜「手机」时，直接查倒排表拿到文档列表，而不是扫全表。

### 3.5 写入与可见性：refresh 与 flush

| 阶段 | 说明 |
|------|------|
| 写入 | 进内存 buffer |
| **refresh**（默认 1s） | buffer 写入 segment，**可被搜索**（近实时） |
| **flush** | segment 持久化到磁盘，写 translog |

批量导入时可临时调大 `refresh_interval` 提升写入性能。

---

<h2 id="sec-04-architecture">4. ES 架构：单机 vs 集群</h2>

### 4.1 单机模式

适合：本地开发、功能验证、小数据量。

```
┌─────────────────────────┐
│  单节点 (1 node)         │
│  - master + data 角色    │
│  - 默认 1 主分片 1 副本   │
└─────────────────────────┘
```

**注意**：单节点时副本无法分配（没有第二台机器），集群状态会 **yellow**（功能正常，只是副本未分配）。

### 4.2 集群模式

适合：生产环境、大数据量、高可用。

```
                    ┌──────────┐
                    │ Node 1   │  master eligible + data
                    └────┬─────┘
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
      ┌─────────┐  ┌─────────┐  ┌─────────┐
      │ Node 2  │  │ Node 3  │  │ Node 4  │
      │  data   │  │  data   │  │  data   │
      └─────────┘  └─────────┘  └─────────┘
```

**分片规划经验**：

- 主分片数创建索引时确定，之后只能 **reindex** 修改
- 单个分片建议 **20GB～50GB**（视场景）
- 副本数可以动态调整
- 3 节点集群常见配置：3 主分片 + 1 副本 = 每节点约 2 个分片

### 4.3 集群健康状态

| 状态 | 含义 |
|------|------|
| **green** | 所有主分片、副本分片都正常 |
| **yellow** | 主分片 OK，部分副本未分配（常见于单节点） |
| **red** | 部分主分片丢失，数据可能不可用 |

```bash
curl http://localhost:9200/_cluster/health?pretty
```

---

<h2 id="sec-05-docker">5. Docker 部署 ES（开发/测试）</h2>

### 5.1 单机 Docker Compose（推荐入门）

创建 `docker-compose.yml`：

```yaml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: es-single
    environment:
      - discovery.type=single-node          # 单节点模式
      - xpack.security.enabled=false          # 开发环境关闭安全（生产勿用）
      - ES_JAVA_OPTS=-Xms512m -Xmx512m        # JVM 堆内存
      - bootstrap.memory_lock=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
    ports:
      - "9200:9200"   # HTTP API
      - "9300:9300"   # 节点间通信
    volumes:
      - es-data:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

volumes:
  es-data:
```

启动：

```bash
docker compose up -d
curl http://localhost:9200    # 看到 JSON 版本信息即成功
```

浏览器打开 Kibana：`http://localhost:5601`

### 5.2 三节点集群 Docker Compose（模拟生产）

```yaml
version: '3.8'
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - node.name=es01
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es02,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ports: ["9200:9200"]
    volumes: [es01-data:/usr/share/elasticsearch/data]

  es02:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    volumes: [es02-data:/usr/share/elasticsearch/data]

  es03:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    volumes: [es03-data:/usr/share/elasticsearch/data]

volumes:
  es01-data:
  es02-data:
  es03-data:
```

验证：

```bash
curl http://localhost:9200/_cat/nodes?v
curl http://localhost:9200/_cluster/health?pretty   # 应 green
```

### 5.3 Docker 常见问题

| 问题 | 解决 |
|------|------|
| `max virtual memory areas vm.max_map_count [65530] is too low` | `sysctl -w vm.max_map_count=262144` |
| 容器 OOM | 增大 Docker 内存，或调低 `ES_JAVA_OPTS` |
| 权限错误 | `chown 1000:1000` 数据目录 |
| ES 8 默认 HTTPS | 开发可关 security，或配置证书 |

### 5.4 安装 IK 中文分词（Docker）

```bash
# 进入容器安装（版本需与 ES 一致）
docker exec -it es-single elasticsearch-plugin install \
  https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v8.11.0/elasticsearch-analysis-ik-8.11.0.zip

docker restart es-single
```

测试分词：

```bash
curl -X POST "localhost:9200/_analyze" -H 'Content-Type: application/json' -d'
{
  "analyzer": "ik_max_word",
  "text": "中华人民共和国"
}'
```

---

<h2 id="sec-06-cluster">6. 集群部署要点（生产）</h2>

### 6.1 节点角色分离（中大型集群）

| 角色 | 职责 |
|------|------|
| **master** | 集群管理、选主，不存数据（专用 master 节点） |
| **data** | 存数据、执行查询 |
| **ingest** | 预处理 pipeline |
| **coordinating** | 仅协调请求（可选） |

小集群可以 **master + data 合一**；生产建议至少 3 个 master-eligible 节点（防脑裂）。

### 6.2 关键配置项

```yaml
# elasticsearch.yml 片段
cluster.name: prod-es-cluster
node.name: es-data-01
node.roles: [ data, ingest ]

network.host: 0.0.0.0
http.port: 9200

# 首次集群 bootstrap（仅第一次，之后删除）
# cluster.initial_master_nodes: ["es-01", "es-02", "es-03"]
discovery.seed_hosts: ["es-01:9300", "es-02:9300", "es-03:9300"]

# 内存
bootstrap.memory_lock: true

# 磁盘水位（默认）
# cluster.routing.allocation.disk.watermark.low: 85%
# cluster.routing.allocation.disk.watermark.high: 90%
```

### 6.3 生产安全（ES 8 默认开启）

- 启用 **TLS** 与 **用户认证**
- 创建角色分离的用户（读写分离）
- Kibana 通过 service account 连接
- 不要将 9200 暴露公网

### 6.4 索引生命周期（ILM）

日志类数据可按时间滚动：

```
热节点（SSD，7天）→ 温节点（HDD，30天）→ 冷节点（归档）→ 删除
```

避免单个索引无限增大，便于管理与删除历史数据。

---

<h2 id="sec-07-springboot3">7. Spring Boot 3 集成 ES</h2>

### 7.1 依赖选择

Spring Boot 3 推荐使用官方 **Elasticsearch Java API Client**（基于 REST，替代已废弃的 High Level REST Client）。

**Maven**：

```xml
<properties>
    <java.version>17</java.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
    </dependency>
</dependencies>
```

Spring Boot 3.2+ 会自动引入 `co.elastic.clients:elasticsearch-java`。

### 7.2 配置文件

```yaml
# application.yml
spring:
  elasticsearch:
    uris: http://localhost:9200
    # ES 8 开启安全时：
    # username: elastic
    # password: your-password
    connection-timeout: 5s
    socket-timeout: 30s
```

### 7.3 定义文档实体

```java
@Document(indexName = "products")
public class Product {

    @Id
    private String id;

    @Field(type = FieldType.Text, analyzer = "ik_max_word", searchAnalyzer = "ik_smart")
    private String title;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Double)
    private Double price;

    @Field(type = FieldType.Integer)
    private Integer stock;

    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second)
    private LocalDateTime createTime;

    // getter / setter
}
```

### 7.4 Repository 方式（简单 CRUD + 基础搜索）

```java
public interface ProductRepository extends ElasticsearchRepository<Product, String> {

    List<Product> findByTitleContaining(String keyword);

    List<Product> findByCategoryAndPriceBetween(String category, Double min, Double max);
}
```

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;
    private final ElasticsearchOperations elasticsearchOperations;

    public Product save(Product product) {
        return productRepository.save(product);
    }

    public void deleteById(String id) {
        productRepository.deleteById(id);
    }

    public Iterable<Product> searchByKeyword(String keyword) {
        return productRepository.findByTitleContaining(keyword);
    }
}
```

### 7.5 原生 Client 复杂查询（推荐掌握）

```java
@Configuration
public class ElasticsearchConfig {

    @Bean
    public ElasticsearchClient elasticsearchClient(RestClient restClient) {
        RestClientTransport transport = new RestClientTransport(
            restClient, new JacksonJsonpMapper());
        return new ElasticsearchClient(transport);
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchClient client;

    public SearchResponse<Product> search(String keyword, int page, int size) throws IOException {
        return client.search(s -> s
            .index("products")
            .from(page * size)
            .size(size)
            .query(q -> q
                .bool(b -> b
                    .must(m -> m
                        .multiMatch(mm -> mm
                            .fields("title", "description")
                            .query(keyword)
                        )
                    )
                    .filter(f -> f
                        .range(r -> r
                            .field("price")
                            .gte(JsonData.of(0))
                            .lte(JsonData.of(9999))
                        )
                    )
                )
            )
            .sort(so -> so.field(f -> f.field("price").order(SortOrder.Asc)))
        , Product.class);
    }
}
```

### 7.6 索引创建（启动时自动或手动）

**方式一：自动**（Spring Data 可 `@Document` + `createIndex = true`）

**方式二：手动 PUT Mapping**（生产推荐，可控）

```bash
curl -X PUT "localhost:9200/products" -H 'Content-Type: application/json' -d'
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "default": { "type": "ik_max_word" }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": { "type": "text", "analyzer": "ik_max_word" },
      "category": { "type": "keyword" },
      "price": { "type": "double" },
      "createTime": { "type": "date" }
    }
  }
}'
```

### 7.7 数据同步策略（MySQL → ES）

| 方案 | 说明 |
|------|------|
| 双写 | 写 MySQL 同时写 ES，简单但一致性难保证 |
| 异步 MQ | 写 MySQL 后发消息，消费者更新 ES（**推荐**） |
| Canal / Debezium | 监听 binlog 同步（近实时） |
| 定时全量 | 适合要求不高的场景 |

**原则**：MySQL 是数据源，ES 是搜索视图，允许短暂延迟。

---

<h2 id="sec-08-apis">8. 常用 API 详解</h2>

> ES 是 RESTful API，以下用 `curl` 演示。Spring Boot 中对应 `ElasticsearchClient` 的方法名类似。

### 8.1 集群与节点

| API | 方法 | 说明 |
|-----|------|------|
| 集群健康 | `GET /_cluster/health` | green/yellow/red |
| 集群状态 | `GET /_cluster/state` | 详细元数据 |
| 节点列表 | `GET /_cat/nodes?v` | 人类可读 |
| 节点信息 | `GET /_nodes` | JVM、插件等 |
| 集群设置 | `PUT /_cluster/settings` | 动态调整部分参数 |

```bash
curl "localhost:9200/_cluster/health?pretty"
curl "localhost:9200/_cat/nodes?v&h=name,heap.percent,cpu,load_1m,node.role"
```

---

### 8.2 索引管理

| API | 方法 | 说明 |
|-----|------|------|
| 创建索引 | `PUT /{index}` | 含 settings、mappings |
| 删除索引 | `DELETE /{index}` | 慎用 |
| 查看索引 | `GET /{index}` | settings + mappings |
| 索引列表 | `GET /_cat/indices?v` | 大小、文档数 |
| 关闭/打开 | `POST /{index}/_close` | 释放资源 |
| 别名 | `POST /_aliases` | 零停机 reindex |
| 刷新 | `POST /{index}/_refresh` | 手动 refresh |
| 强制合并 | `POST /{index}/_forcemerge` | 合并 segment，低峰执行 |

```bash
# 创建索引
curl -X PUT "localhost:9200/my-index"

# 查看所有索引
curl "localhost:9200/_cat/indices?v&s=store.size:desc"

# 别名切换（蓝绿发布）
curl -X POST "localhost:9200/_aliases" -H 'Content-Type: application/json' -d'
{
  "actions": [
    { "remove": { "index": "products_v1", "alias": "products" } },
    { "add":    { "index": "products_v2", "alias": "products" } }
  ]
}'
```

---

### 8.3 文档 CRUD

| API | 方法 | 说明 |
|-----|------|------|
| 创建/覆盖 | `PUT /{index}/_doc/{id}` | 指定 ID 全量写 |
| 创建（自动生成 ID） | `POST /{index}/_doc` | ES 生成 _id |
| 部分更新 | `POST /{index}/_update/{id}` | doc 局部字段 |
| 获取 | `GET /{index}/_doc/{id}` | 按 ID 查 |
| 删除 | `DELETE /{index}/_doc/{id}` | 按 ID 删 |
| 存在判断 | `HEAD /{index}/_doc/{id}` | 返回 200/404 |

```bash
# 写入文档
curl -X PUT "localhost:9200/products/_doc/1" -H 'Content-Type: application/json' -d'
{
  "title": "Apple iPhone 15",
  "category": "phone",
  "price": 5999
}'

# 部分更新
curl -X POST "localhost:9200/products/_update/1" -H 'Content-Type: application/json' -d'
{
  "doc": { "price": 5499 }
}'

# 读取
curl "localhost:9200/products/_doc/1?pretty"
```

---

### 8.4 搜索 API（核心）

| API | 方法 | 说明 |
|-----|------|------|
| 搜索 | `GET/POST /{index}/_search` | 主搜索入口 |
| 滚动查询 | `POST /_search?scroll=1m` | 大批量导出 |
| Point in Time | `POST /{index}/_pit` | 深分页（推荐替代 scroll） |
| Search After | 搜索 body 带 `search_after` | 深分页 |
| 计数 | `GET /{index}/_count` | 只返回总数 |
| 解释评分 | `GET /{index}/_explain/{id}` | 调试相关度 |

#### 8.4.1 match 查询（全文）

```bash
curl -X POST "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "title": "苹果手机"
    }
  }
}'
```

#### 8.4.2 term 查询（精确）

```bash
curl -X POST "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "term": {
      "category.keyword": "phone"
    }
  }
}'
```

#### 8.4.3 bool 组合（必会）

```bash
curl -X POST "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "手机" } }
      ],
      "filter": [
        { "term": { "category": "phone" } },
        { "range": { "price": { "gte": 1000, "lte": 8000 } } }
      ],
      "must_not": [
        { "term": { "status": "offline" } }
      ],
      "should": [
        { "term": { "tags": "5g" } }
      ],
      "minimum_should_match": 1
    }
  },
  "from": 0,
  "size": 10,
  "sort": [{ "price": "asc" }],
  "_source": ["title", "price"]
}'
```

| bool 子句 | 作用 | 是否算分 |
|-----------|------|----------|
| must | 必须满足 | ✅ |
| filter | 必须满足 | ❌（有 cache，更快） |
| should | 满足更好 | ✅ |
| must_not | 必须不满足 | ❌ |

#### 8.4.4 聚合（Agg）

```bash
curl -X POST "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": { "field": "category", "size": 10 },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }
      }
    }
  }
}'
```

常用聚合：`terms`（分组）、`avg/sum/min/max`、`date_histogram`（按天统计）、`cardinality`（去重计数）。

---

### 8.5 批量操作 Bulk（生产必用）

| API | 方法 | 说明 |
|-----|------|------|
| 批量写入 | `POST /_bulk` | 增删改批量 |
| 按索引批量 | `POST /{index}/_bulk` | 指定索引 |

```bash
curl -X POST "localhost:9200/_bulk" -H 'Content-Type: application/json' -d'
{ "index": { "_index": "products", "_id": "1" } }
{ "title": "商品A", "price": 100 }
{ "index": { "_index": "products", "_id": "2" } }
{ "title": "商品B", "price": 200 }
{ "delete": { "_index": "products", "_id": "3" } }
'
```

**最佳实践**：

- 每批 **1000～5000** 条或 **5～15MB**
- 批量导入时设 `refresh_interval=-1`，完成后恢复
- 检查响应里 `errors: true` 并处理失败项

---

### 8.6 Reindex（索引迁移/改 mapping）

```bash
curl -X POST "localhost:9200/_reindex" -H 'Content-Type: application/json' -d'
{
  "source": { "index": "products_v1" },
  "dest":   { "index": "products_v2" }
}'
```

用于：修改主分片数、改 mapping、升级数据结构。

---

### 8.7 Ingest Pipeline（预处理）

```bash
# 创建 pipeline：添加处理时间戳
curl -X PUT "localhost:9200/_ingest/pipeline/add_timestamp" -H 'Content-Type: application/json' -d'
{
  "description": "add processed_at",
  "processors": [
    { "set": { "field": "processed_at", "value": "{{_ingest.timestamp}}" } }
  ]
}'

# 写入时指定 pipeline
curl -X POST "localhost:9200/logs/_doc?pipeline=add_timestamp" -H 'Content-Type: application/json' -d'
{ "message": "user login" }'
```

---

### 8.8 API 速查表

| 分类 | 常用端点 |
|------|----------|
| 集群 | `_cluster/health`, `_cluster/stats`, `_cat/nodes` |
| 索引 | `PUT/GET/DELETE /{index}`, `_cat/indices`, `_aliases` |
| 文档 | `_doc/{id}`, `_update/{id}`, `_bulk` |
| 搜索 | `_search`, `_count`, `_validate/query` |
| 分析 | `_analyze`, `_field_caps` |
| 运维 | `_tasks`, `_cluster/reroute`, `_cat/shards` |

---

<h2 id="sec-09-pitfalls">9. 使用中的常见坑与防范</h2>

### 坑 1：text 字段做聚合/排序

**现象**：对 `title`（text）做 terms 聚合报错或结果异常。

**原因**：text 会分词，不适合精确聚合。

**防范**：聚合、排序用 `keyword` 或 `title.keyword`。

---

### 坑 2：深度分页 `from + size` 过大

**现象**：`from=100000, size=10` 极慢，内存暴涨。

**原因**：每个分片都要排序并返回 from+size 条，协调节点再合并。

**防范**：

- 业务分页限制最大页数（如 100 页）
- 深分页用 **search_after** 或 **PIT + search_after**
- 导出用 **scroll**（注意及时 clear scroll）

---

### 坑 3：Mapping 创建后难以修改字段类型

**现象**：把 `price` 从 `integer` 改成 `double` 直接 update mapping 失败。

**原因**：已存在数据的字段类型不可随意改。

**防范**：

- 上线前设计好 mapping
- 变更用 **新索引 + reindex + 别名切换**
- 开发环境用 index template 统一规范

---

### 坑 4：主分片数设错

**现象**：索引 500GB 只有 1 个主分片，扩容节点也无法拆分。

**原因**：主分片数创建时固定。

**防范**：

- 按数据量预估：`分片数 ≈ 数据量 / 30GB`
- 日志按天/按大小 rollover
- 测试环境也要养成正确习惯

---

### 坑 5：单节点 yellow 误以为故障

**现象**：健康状态 yellow，以为 ES 坏了。

**原因**：副本分片无法分配到同一节点（正常）。

**防范**：开发环境可设 `number_of_replicas: 0`；生产至少 2 节点。

---

### 坑 6：频繁 update 同一文档

**现象**：写入 QPS 不高但 CPU、磁盘 IO 很高。

**原因**：update 底层是 read-modify-write，产生大量 segment 和 tombstone。

**防范**：

- 减少更新频率，批量 merge 更新
- 日志类数据只 append 不 update
- 用 `_version` 或外部版本号控制并发

---

### 坑 7：Bulk 单批过大或过小

**现象**：过大 OOM/超时；过小吞吐差。

**防范**：每批 5～15MB，观察 `_bulk` 耗时与 `429` 拒绝，动态调整；遇到 **429** 退避重试。

---

### 坑 8：不设堆内存上限或超过物理内存

**现象**：ES 进程被 OOM Kill，或触发 swap 性能骤降。

**防范**：

- `ES_JAVA_OPTS=-Xms4g -Xmx4g`（**Xms = Xmx**）
- 堆 ≤ 物理内存 50%，且 ≤ **32GB**（避免 Compressed OOP 失效）
- `bootstrap.memory_lock: true` 锁内存

---

### 坑 9：磁盘满了仍写入

**现象**：集群 red，分片无法分配。

**防范**：

- 监控磁盘使用率
- 理解 watermark：85% 不再迁_shard 进来，90% 尝试迁出，95% 索引只读
- ILM 自动删旧索引

---

### 坑 10：Spring Data 版本与 ES 版本不匹配

**现象**：启动报错、序列化异常、API 找不到。

**防范**：

- Spring Boot BOM 管理版本，勿手动乱引 ES client
- 查 [Spring Boot 官方文档](https://docs.spring.io/spring-boot/docs/current/reference/html/data.html#data.nosql.elasticsearch) 对应 ES 版本
- ES 8 服务器 + Boot 3.2+ 客户端是常见组合

---

### 坑 11：MySQL 与 ES 数据不一致

**现象**：DB 已删，搜索还能搜到。

**防范**：

- 以 MySQL 为准，MQ 同步 ES 带重试与死信
- 定期对账任务（抽样比对）
- 删除走统一服务，同时删 DB 和 ES

---

### 坑 12：查询写进 must 而非 filter

**现象**：缓存不生效，重复 filter 也慢。

**防范**：不需要评分的条件（状态、时间范围）放 **filter**。

---

### 坑 13：wildcard 前缀通配 `*abc` 或 `%abc%` 式滥用

**现象**：全表扫描级慢查询。

**防范**：用 ngram/edge_ngram 或 ES 的 `search_as_you_type`；避免 leading wildcard。

---

### 坑 14：生产关闭 security

**现象**：9200 暴露公网被删库、挖矿。

**防范**：开启 xpack security、网络隔离、最小权限账号、禁止公网暴露。

---

<h2 id="sec-10-monitoring">10. 监控工具与使用方式</h2>

### 10.1 监控什么

| 维度 | 关键指标 |
|------|----------|
| **集群** | health、pending tasks、unassigned shards |
| **节点** | JVM heap、GC、CPU、load、disk |
| **索引** | 文档数、store size、query/index 耗时 |
| **搜索** | QPS、P99 延迟、慢查询 |
| **写入** | indexing rate、bulk 拒绝数 |

---

### 10.2 Kibana（官方，首选可视化）

**用途**：Dev Tools 调试 DSL、Index Management、监控大盘、日志 Discover。

**访问**：`http://localhost:5601`

**Dev Tools 示例**：

```json
GET /products/_search
{
  "query": { "match_all": {} },
  "size": 5
}
```

**Stack Monitoring**（需 X-Pack）：监控 ES、Kibana、Logstash 自身指标。

---

### 10.3 _cat API（命令行快速看）

```bash
# 索引大小排序
curl "localhost:9200/_cat/indices?v&s=store.size:desc"

# 分片分布
curl "localhost:9200/_cat/shards?v"

# 进行中的任务
curl "localhost:9200/_cat/tasks?v"

# 线程池（看 reject）
curl "localhost:9200/_cat/thread_pool?v&h=name,active,rejected,completed"
```

---

### 10.4 Prometheus + Elasticsearch Exporter

**架构**：

```
ES → elasticsearch-exporter (:9114) → Prometheus → Grafana
```

**docker 运行 exporter**：

```bash
docker run -d --name es-exporter \
  -p 9114:9114 \
  quay.io/prometheuscommunity/elasticsearch-exporter:latest \
  --es.uri=http://host.docker.internal:9200
```

**Grafana**：导入社区 Dashboard（如 ID `14191` Elasticsearch Exporter Quickstart）。

**告警示例**：

- `elasticsearch_cluster_health_status != 0`（非 green）
- `elasticsearch_jvm_memory_used_bytes / max > 0.85`
- `elasticsearch_thread_pool_rejected_total` 持续增长

---

### 10.5 Elastic Stack 监控（Metricbeat）

```yaml
# metricbeat.yml 片段
metricbeat.modules:
  - module: elasticsearch
    metricsets: [node, node_stats, cluster_stats, index, index_recovery, shard]
    period: 10s
    hosts: ["http://localhost:9200"]
```

配合 Kibana **Metrics** 应用查看趋势。

---

### 10.6 慢查询日志

```yaml
# elasticsearch.yml
index.search.slowlog.threshold.query.warn: 2s
index.search.slowlog.threshold.fetch.warn: 500ms
index.indexing.slowlog.threshold.index.warn: 5s
```

或在索引 settings 动态设置：

```bash
curl -X PUT "localhost:9200/products/_settings" -H 'Content-Type: application/json' -d'
{
  "index.search.slowlog.threshold.query.warn": "1s"
}'
```

日志路径：`logs/{cluster-name}_index_search_slowlog.log`

---

### 10.7 Java 应用侧监控

Spring Boot Actuator + Micrometer：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

对 ES 客户端调用可：

- 记录每次搜索耗时（Micrometer Timer）
- 慢查询 > 500ms 打 warn 日志
- 统计 bulk 成功率

---

### 10.8 常用诊断命令汇总

```bash
# 集群为什么 yellow/red
curl "localhost:9200/_cluster/allocation/explain?pretty"

# 查看热点线程
curl "localhost:9200/_nodes/hot_threads"

# 查看 pending 任务（集群变更卡住）
curl "localhost:9200/_cluster/pending_tasks?pretty"

# Profile 分析慢查询
curl -X POST "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
{
  "profile": true,
  "query": { "match": { "title": "手机" } }
}'
```

---

### 10.9 监控工具选型建议

| 场景 | 推荐 |
|------|------|
| 开发调试 | Kibana Dev Tools |
| 小规模/自建 | Prometheus + Grafana + exporter |
| Elastic 商业订阅 | Elastic Observability（APM + Logs + Metrics 一体） |
| 告警 | Alertmanager / 钉钉 Webhook / PagerDuty |

---

<h2 id="sec-11-cheatsheet">11. 学习路径与速查</h2>

### 11.1 推荐学习顺序

```
1. 理解概念（Index/Document/Mapping/Shard）
      ↓
2. Docker 单机跑起来 + Kibana Dev Tools 练 DSL
      ↓
3. 掌握 CRUD + bool 查询 + 聚合
      ↓
4. Spring Boot 3 集成 + 业务搜索接口
      ↓
5. 批量导入、reindex、别名
      ↓
6. 集群部署、监控、慢查询、避坑
```

### 11.2 一图总结

```
        ┌─────────────┐
        │   业务应用   │  Spring Boot 3
        └──────┬──────┘
               │ REST / Java Client
        ┌──────▼──────┐
        │ Elasticsearch│  搜索 + 聚合
        │   Cluster    │
        └──────┬──────┘
               │
    ┌──────────┼──────────┐
    ▼          ▼          ▼
 MySQL      Redis      MQ
 (主数据)   (缓存)   (同步ES)
```

### 11.3 锚点 ID 对照表

| 章节 | 锚点 ID |
|------|---------|
| 1 | `sec-01-why-es` |
| 2 | `sec-02-scenarios` |
| 3 | `sec-03-concepts` |
| 4 | `sec-04-architecture` |
| 5 | `sec-05-docker` |
| 6 | `sec-06-cluster` |
| 7 | `sec-07-springboot3` |
| 8 | `sec-08-apis` |
| 9 | `sec-09-pitfalls` |
| 10 | `sec-10-monitoring` |
| 11 | `sec-11-cheatsheet` |

---

## 附录：延伸阅读

- [Elasticsearch 官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Spring Data Elasticsearch](https://docs.spring.io/spring-data/elasticsearch/reference/)
- [IK 分词器](https://github.com/medcl/elasticsearch-analysis-ik)

---

> 文档路径：`搜索引擎/Elasticsearch实践指南.md`
