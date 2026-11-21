# DocForge

DocForge 是一个面向企业知识库的 RAG 应用：把文档上传、解析、切片、向量化、混合检索和带引用的问答串成一条可观测的流水线。后端 Spring Boot 3 + Vue 3 前端，检索侧同时用 BM25 与向量召回，回答由大模型基于命中的片段生成并回填引用。

## 核心能力

- **文档全链路**：分片上传 → 合并入对象存储 → LiteParse 解析（PDF/图片 OCR）→ 语义切片 → 向量化 → 写入 Elasticsearch，每一步都有状态可查、失败可重试。
- **混合检索**：关键词（BM25）与向量（kNN）双路召回后融合排序，命中片段带父子层级，可回溯到原文位置。
- **带引用的对话**：WebSocket 流式返回，回答里的引用映射在会话历史中持久化，重新打开历史仍能看到出处。
- **多租户与权限**：按组织标签隔离数据，文档分公开/私有，管理员可按用户与组织维度查看。
- **异步处理**：文件处理走 Kafka，解析与向量化不阻塞上传请求，失败进死信队列。
- **离线评测**：内置 50 条标注问答样本，可对检索与回答质量做回归。

## 技术栈

| 层 | 选型 |
| --- | --- |
| 后端 | Spring Boot 3.4.2 / Java 17 / Spring Security + JWT / WebFlux |
| 存储 | MySQL 8（JPA）/ Redis / MinIO |
| 检索 | Elasticsearch 8.10（IK 中文分词，BM25 + kNN 混合） |
| 消息 | Apache Kafka |
| 文档解析 | LiteParse CLI（含 Tesseract OCR） |
| 模型 | DeepSeek（对话）/ text-embedding-v4（2048 维向量） |
| 前端 | Vue 3 + TypeScript + Vite + Naive UI + Pinia + UnoCSS |

## 架构

```
浏览器 ──Vue 3──> Vite dev proxy ──> Spring Boot :8081
                                      ├── MySQL        会话 / 文档元数据 / 切片
                                      ├── Redis        会话上下文 / 限流
                                      ├── MinIO        原始文件
                                      ├── Kafka        文件处理流水线
                                      └── Elasticsearch 向量 + 全文索引
                                            └── DeepSeek / Embedding API
```

## 目录结构

```
src/main/java/com/docforge/
├── client/       外部模型与解析客户端
├── config/       安全、JWT、模型参数等配置
├── consumer/     Kafka 消费端
├── controller/   REST 接口
├── entity/       JPA 实体
├── handler/      WebSocket 处理器
├── repository/   数据访问
├── service/      业务逻辑（解析 / 切片 / 向量化 / 检索 / 对话）
└── utils/
frontend/         Vue 3 前端（dev 端口 9527）
docs/             设计与评测文档
```

## 快速开始

### 前置

Java 17、Maven 3.8+、Node 18.20+、pnpm 8.7+，以及本机可用的 MySQL 8、Redis、Elasticsearch 8.10（装 IK 插件）、Kafka、MinIO。

### 1. 配置环境变量

仓库根目录放一个 `.env`，启动时会被注入为配置源：

```dotenv
SPRING_PROFILES_ACTIVE=dev
SERVER_PORT=8081

SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/docforge?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
SPRING_DATASOURCE_USERNAME=root
SPRING_DATASOURCE_PASSWORD=123456

SPRING_DATA_REDIS_HOST=localhost
SPRING_DATA_REDIS_PORT=6379

SPRING_ELASTICSEARCH_URIS=https://localhost:9200
SPRING_ELASTICSEARCH_USERNAME=elastic
SPRING_ELASTICSEARCH_PASSWORD=changeme
ELASTICSEARCH_INSECURE_TRUST_ALL=true

KAFKA_BOOTSTRAP_SERVERS=127.0.0.1:9092
MINIO_ENDPOINT=http://localhost:9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_BUCKET=uploads

DEEPSEEK_API_KEY=
EMBEDDING_API_KEY=
EMBEDDING_API_MODEL=text-embedding-v4
EMBEDDING_DIMENSION=2048
JWT_SECRET_KEY=
```

### 2. 建库与索引

```sql
CREATE DATABASE IF NOT EXISTS docforge DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci;
```

首次启动会自动建表；知识库索引在文档向量化时按 `knowledge_base_<模型>_<维度>` 创建。

### 3. 启动

```bash
# 后端
mvn spring-boot:run

# 前端
cd frontend && pnpm install && pnpm dev   # http://localhost:9527
```

## 主要接口

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/v1/users/login` | 登录，返回 accessToken / refreshToken |
| POST | `/api/v1/upload/chunk` | 分片上传 |
| POST | `/api/v1/upload/merge` | 合并分片并触发处理流水线 |
| POST | `/api/v1/parse` | 单独解析文档 |
| GET | `/api/v1/search/hybrid` | 混合检索 |
| DELETE | `/api/v1/documents/{fileMd5}` | 删除文档及其索引 |
| POST | `/api/v1/documents/{fileMd5}/reindex` | 重建索引 |
| WS | `/api/v1/chat/websocket-token` | 获取对话通道令牌 |

## 评测

`src/main/resources/eval/docforge-eval-v1.jsonl` 提供带标注的问答集，覆盖检索命中与引用正确性；`mvn test` 会跑解析器与检索链路的单测。

## 说明

- 文档解析依赖本机 `lit` 命令（`pip install liteparse`），未安装时上传会失败。
- Elasticsearch 单节点下索引健康值为 yellow 属正常（副本无法分配）。
- 模型维度和索引名绑定，换 Embedding 模型需要重建索引。
