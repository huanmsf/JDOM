# MinIO 详解 - 从零到精通

> 版本：MinIO RELEASE.2024+ | Java SDK 8.5.x | Spring Boot 3.x

---

## 目录

- [Part 1: MinIO 整体架构](#part-1-minio-整体架构)
- [Part 2: MinIO 存储原理](#part-2-minio-存储原理)
- [Part 3: MinIO 部署与配置](#part-3-minio-部署与配置)
- [Part 4: Bucket 与对象管理](#part-4-bucket-与对象管理)
- [Part 5: 权限与安全](#part-5-权限与安全)
- [Part 6: 通知与事件](#part-6-通知与事件)
- [Part 7: Spring Boot 集成](#part-7-spring-boot-集成)
- [Part 8: 实战案例](#part-8-实战案例)
- [Part 9: 性能调优与运维](#part-9-性能调优与运维)
- [Part 10: 面试题 FAQ](#part-10-面试题-faq)

---

# Part 1: MinIO 整体架构

## 1.1 MinIO 是什么

MinIO 是一款高性能、开源的对象存储系统，完全兼容 Amazon S3 API，专为云原生、AI/ML、数据湖和数据库备份而设计。MinIO 以极简主义著称，仅需一个可执行文件即可运行，却能提供企业级的高可靠性。

MinIO 于 2014 年创建，使用 Go 语言编写，已开源至 GitHub（Apache 2.0 许可证），核心理念是"软件定义存储"，可在标准 x86 服务器上构建企业级对象存储集群。

```
+------------------------------------------------------------------+
|                    MinIO 定位全景图                              |
+------------------------------------------------------------------+
|                                                                  |
|   应用层:  AI/ML  |  数据湖  |  备份归档  |  云原生应用          |
|           --------|----------|------------|----------            |
|                        S3 API 兼容层                            |
|           ----------------------------------------              |
|             |         MinIO 存储引擎              |             |
|             |  纠删码 | 位衰减保护 | 内联EC       |             |
|           ----------------------------------------              |
|   硬件层:    HDD / SSD / NVMe / 网络文件系统                    |
|                                                                  |
+------------------------------------------------------------------+
```

## 1.2 MinIO 核心特性

| 特性 | 描述 |
|------|------|
| S3 兼容 | 100% 兼容 Amazon S3 API，无需修改代码即可迁移 |
| 高性能 | 单节点读写速度可达 325 GB/s 读、100 GB/s 写 |
| 纠删码 | 基于 Reed-Solomon 算法，支持最多 N/2 节点故障 |
| 极简部署 | 单个 Go 二进制文件，零外部依赖 |
| 多云支持 | 可在任何云、裸机、边缘设备上运行 |
| 数据保护 | 支持 SSE（服务端加密）、TLS、Vault 集成 |
| 版本控制 | 支持对象版本管理，防止意外删除 |
| 生命周期 | 自动化对象生命周期管理（ILM） |

## 1.3 MinIO vs 竞品对比

```
+---------------------+----------+----------+----------+----------+
| 特性                | MinIO    | Ceph RGW | Swift    | S3       |
+---------------------+----------+----------+----------+----------+
| S3 兼容性           | 完全     | 部分     | 部分     | 原生     |
| 部署复杂度          | 极低     | 极高     | 高       | 托管     |
| 性能（读）          | 极高     | 中       | 中       | 高       |
| 最小部署规模        | 1 节点   | 3+ 节点  | 3+ 节点  | 托管     |
| 开源协议            | Apache2  | LGPL     | Apache2  | 商业     |
| 内存占用            | 极低     | 高       | 中       | N/A      |
| 纠删码              | 内置     | 内置     | 无       | 内置     |
+---------------------+----------+----------+----------+----------+
```

## 1.4 MinIO 架构模式

MinIO 支持三种核心部署架构：

### 1.4.1 单节点单驱动（SNSD）

最简单的部署方式，适用于开发测试环境。单个服务器，单个磁盘，无冗余。

```
+--------------------+
|   MinIO Server     |
|  +--------------+  |
|  |   /data      |  |
|  | (单个磁盘)   |  |
|  +--------------+  |
+--------------------+
  SNSD: 无高可用，仅开发测试
```

### 1.4.2 单节点多驱动（SNMD）

单个服务器，多个磁盘，通过纠删码提供数据保护。

```
+------------------------------------------------+
|                 MinIO Server                   |
|  +------+ +------+ +------+ +------+ +------+ |
|  |/data1| |/data2| |/data3| |/data4| |/data5| |
|  +------+ +------+ +------+ +------+ +------+ |
+------------------------------------------------+
  SNMD: 单节点高可用，支持 N/2 磁盘故障
```

### 1.4.3 多节点多驱动（MNMD）

多个服务器，每台多个磁盘，生产推荐部署方式。

```
+----------+  +----------+  +----------+  +----------+
|  Node 1  |  |  Node 2  |  |  Node 3  |  |  Node 4  |
| /d1.../d4|  | /d1.../d4|  | /d1.../d4|  | /d1.../d4|
+----+-----+  +----+-----+  +----+-----+  +----+-----+
     |             |              |              |
     +-------------+--------------+--------------+
                   分布式纠删码集群
         (MNMD: 最高可用性，生产推荐)
```

## 1.5 MinIO 数据流架构

```
+------------------+     HTTP/HTTPS      +--------------------+
|   客户端          |-------------------->|   MinIO Gateway    |
|  (SDK/mc/curl)   |<--------------------|   (S3 API Handler) |
+------------------+                     +----------+---------+
                                                    |
                               +--------------------v-----------+
                               |       对象路由层                |
                               |  (一致性哈希 / 纠删码组分配)   |
                               +--------------------+-----------+
                                                    |
              +--------------------+----------------v-----------+
              |                    |                            |
    +---------v-------+ +----------v------+ +---------v-------+
    |   磁盘组 1      | |   磁盘组 2      | |   磁盘组 3      |
    | (纠删码条带 1)  | | (纠删码条带 2)  | | (纠删码条带 3)  |
    +-----------------+ +-----------------+ +-----------------+
```

---

# Part 2: MinIO 存储原理

## 2.1 纠删码（Erasure Coding）

纠删码是 MinIO 数据保护的核心机制，基于 Reed-Solomon 算法实现。

### 2.1.1 基本原理

MinIO 将数据分成数据块（Data Shards）和奇偶校验块（Parity Shards）。

```
对象: "video.mp4" (100MB)
纠删码组: 4 数据块 + 4 奇偶校验块 (EC:4)

+--------+--------+--------+--------+  +--------+--------+--------+--------+
| 数据块1 | 数据块2 | 数据块3 | 数据块4 |  | 校验块1 | 校验块2 | 校验块3 | 校验块4 |
| 25MB   | 25MB   | 25MB   | 25MB   |  | 25MB   | 25MB   | 25MB   | 25MB   |
+--------+--------+--------+--------+  +--------+--------+--------+--------+
   磁盘1    磁盘2    磁盘3    磁盘4         磁盘5    磁盘6    磁盘7    磁盘8

即使任意 4 块磁盘损坏，仍可完全恢复数据！
```

### 2.1.2 存储开销

| EC 配置 | 数据块 | 校验块 | 最大容忍故障 | 存储开销 |
|---------|--------|--------|-------------|----------|
| EC:2    | N-2    | 2      | 2 块        | N/(N-2)x |
| EC:4    | N-4    | 4      | 4 块        | N/(N-4)x |
| EC:6    | N-6    | 6      | 6 块        | N/(N-6)x |
| EC:8    | N-8    | 8      | 8 块        | N/(N-8)x |

> 注意：MinIO 要求校验块数量 <= 数据块数量（即 EC <= N/2）

### 2.1.3 Reed-Solomon 数学基础

Reed-Solomon 编码基于有限域（Galois Field，GF(2^8)）多项式插值：

```
给定 k 个数据块 d1, d2, ..., dk
构造唯一的 k-1 次多项式 P(x) 满足 P(i) = di
计算 m 个校验块: p1 = P(k+1), p2 = P(k+2), ..., pm = P(k+m)

恢复时：任意已知 k 个值（数据或校验），通过拉格朗日插值即可还原多项式，
进而恢复所有 k 个数据块。
```

## 2.2 位衰减保护（Bitrot Protection）

位衰减（Bitrot）是指存储介质上的数据在静默状态下发生的随机比特翻转错误，传统文件系统无法检测。

MinIO 对每个对象的每个分片计算并存储校验和（HighwayHash 256），读取时自动验证：

```
写入流程:
  对象数据 --> 计算 HighwayHash-256 --> 与数据块一起存储

读取流程:
  读取数据块 --> 重新计算哈希 --> 与存储哈希比对
  匹配   --> 返回数据
  不匹配 --> 标记损坏 --> 从其他分片纠删恢复
```

HighwayHash-256 是 Google 设计的高速非加密哈希算法，利用 SIMD 指令集，速度远快于 SHA-256。

## 2.3 内联纠删码（Inline Erasure Coding）

MinIO 的内联 EC 是指数据写入和纠删码计算同步完成，无需先写再编码的两阶段操作：

```
+-----------------------------+
|       客户端上传数据          |
+-------------+---------------+
              |
              v  (实时流式处理)
+-----------------------------+
|   内联 EC 编码器             |
|  输入: 原始数据流             |
|  输出: k 个数据分片            |
|       + m 个校验分片          |
+----+--------+--------+-------+
     |        |        |
     v        v        v
   磁盘1    磁盘2    磁盘N
```

**优势**：相比先写后编码，内联 EC 减少了一次完整的磁盘 I/O，显著提升写性能。

## 2.4 对象存储格式（XL Meta）

MinIO 使用自定义的 XL Meta 格式存储对象元数据：

```
/data/
  bucket-name/
    object-name/
      xl.meta       <-- 元数据文件 (MessagePack 格式)
      part.1        <-- 数据分片 1
      part.2        <-- 数据分片 2 (多 part 上传时)
```

xl.meta 包含：

- 对象名称、大小、Content-Type
- 纠删码配置（数据块数、校验块数）
- 每个分片的校验和（HighwayHash）
- 版本信息（版本 ID、是否删除标记）
- 自定义元数据（User Metadata）

## 2.5 对象版本控制

开启版本控制后，MinIO 为每个对象版本分配唯一的 UUID Version ID：

```
bucket/object/
  xl.meta   <-- 包含版本链

版本链示例:
  Version 3 (latest): uuid=abc123, size=500KB, modified=2024-01-03
  Version 2:          uuid=def456, size=480KB, modified=2024-01-02
  Version 1:          uuid=ghi789, size=450KB, modified=2024-01-01

删除操作: 在版本链头部插入"删除标记"（Delete Marker），数据实际未删除
彻底删除: 需指定具体 Version ID
```

## 2.6 WORM（Write Once Read Many）

MinIO 通过对象锁（Object Lock）实现 WORM 语义，满足合规要求（SEC 17a-4、HIPAA 等）：

```
+---------------------+          +---------------------+
|  COMPLIANCE 模式    |          |  GOVERNANCE 模式    |
|                     |          |                     |
| 任何人（包括 root） |          | 拥有特殊权限的用户  |
| 都无法删除/修改     |          | 可以绕过锁定        |
| 锁定期内的对象      |          |                     |
+---------------------+          +---------------------+
  合规级别: 最高                   合规级别: 管理员可操作
```

## 2.7 自愈机制（Healing）

MinIO 的后台自愈进程会持续扫描磁盘，检测并修复损坏/丢失的数据：

```
自愈流程:

  1. 扫描器定期扫描每个对象的 xl.meta
  2. 验证所有分片的 HighwayHash
  3. 发现损坏分片 --> 从其他健康分片纠删恢复
  4. 恢复的分片写回原磁盘（或替换磁盘）

手动触发:
  mc admin heal --recursive myminio/
  mc admin heal --recursive myminio/mybucket
```

---

# Part 3: MinIO 部署与配置

## 3.1 快速启动（Docker 单节点）

```bash
# 单节点启动
docker run -d \
  -p 9000:9000 \
  -p 9001:9001 \
  --name minio \
  -v /mnt/data:/data \
  -e "MINIO_ROOT_USER=minioadmin" \
  -e "MINIO_ROOT_PASSWORD=minioadmin" \
  quay.io/minio/minio server /data --console-address ":9001"

# 访问控制台: http://localhost:9001
# 访问 API:   http://localhost:9000
```

## 3.2 Docker Compose 单节点多驱动（SNMD）

```yaml
version: "3.8"

services:
  minio:
    image: quay.io/minio/minio:latest
    container_name: minio
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - ./data1:/data1
      - ./data2:/data2
      - ./data3:/data3
      - ./data4:/data4
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin123
      MINIO_VOLUMES: "/data1 /data2 /data3 /data4"
    command: server --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3
    restart: unless-stopped
```

## 3.3 分布式集群部署（MNMD）

```yaml
version: "3.8"

x-minio-common: &minio-common
  image: quay.io/minio/minio:latest
  environment:
    MINIO_ROOT_USER: minioadmin
    MINIO_ROOT_PASSWORD: minioadmin123
  command: >
    server --console-address ":9001"
    http://minio1:9000/data1 http://minio1:9000/data2
    http://minio2:9000/data1 http://minio2:9000/data2
    http://minio3:9000/data1 http://minio3:9000/data2
    http://minio4:9000/data1 http://minio4:9000/data2

services:
  minio1:
    <<: *minio-common
    hostname: minio1
    volumes:
      - ./data/minio1/data1:/data1
      - ./data/minio1/data2:/data2

  minio2:
    <<: *minio-common
    hostname: minio2
    volumes:
      - ./data/minio2/data1:/data1
      - ./data/minio2/data2:/data2

  minio3:
    <<: *minio-common
    hostname: minio3
    volumes:
      - ./data/minio3/data1:/data1
      - ./data/minio3/data2:/data2

  minio4:
    <<: *minio-common
    hostname: minio4
    volumes:
      - ./data/minio4/data1:/data1
      - ./data/minio4/data2:/data2

  nginx:
    image: nginx:latest
    ports:
      - "9000:9000"
      - "9001:9001"
    depends_on: [minio1, minio2, minio3, minio4]
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
```

## 3.4 Nginx 负载均衡配置

```nginx
upstream minio_api {
    least_conn;
    server minio1:9000;
    server minio2:9000;
    server minio3:9000;
    server minio4:9000;
}

server {
    listen 9000;
    ignore_invalid_headers off;
    client_max_body_size 0;
    proxy_buffering off;
    proxy_request_buffering off;

    location / {
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 300;
        proxy_http_version 1.1;
        chunked_transfer_encoding off;
        proxy_pass http://minio_api;
    }
}
```

## 3.5 mc 客户端工具

mc（MinIO Client）是 MinIO 的命令行工具，兼容 AWS S3。

```bash
# 安装 mc
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && mv mc /usr/local/bin/

# 配置别名
mc alias set myminio http://localhost:9000 minioadmin minioadmin

# 常用命令
mc mb myminio/mybucket              # 创建 Bucket
mc ls myminio/                      # 列出所有 Bucket
mc ls myminio/mybucket              # 列出对象
mc cp /local/file.txt myminio/mybucket/  # 上传文件
mc cp myminio/mybucket/file.txt ./   # 下载文件
mc rm myminio/mybucket/file.txt     # 删除对象
mc mirror /local/dir myminio/mybucket/ # 同步目录
mc stat myminio/mybucket/file.txt   # 查看对象信息

# 管理命令
mc admin info myminio               # 集群信息
mc admin status myminio             # 节点状态
mc admin heal --recursive myminio/  # 自愈修复
mc admin top disks myminio          # 磁盘 I/O 监控
mc admin prometheus generate myminio # 生成 Prometheus 配置
```

## 3.6 存储类（Storage Class）

MinIO 支持通过存储类控制纠删码配置：

```bash
# 设置环境变量（启动时配置）
MINIO_STORAGE_CLASS_STANDARD=EC:4   # 标准存储: 4个校验块
MINIO_STORAGE_CLASS_RRS=EC:2        # 低冗余存储: 2个校验块

# 上传时指定存储类
mc cp --storage-class STANDARD file.txt myminio/bucket/
mc cp --storage-class REDUCED_REDUNDANCY file.txt myminio/bucket/
```

## 3.7 环境变量配置参考

```bash
# 基础配置
MINIO_ROOT_USER=minioadmin          # 管理员用户名
MINIO_ROOT_PASSWORD=minioadmin123   # 管理员密码
MINIO_VOLUMES=/data1 /data2 /data3  # 数据盘路径

# 域名配置
MINIO_DOMAIN=storage.example.com   # 虚拟托管风格域名
MINIO_SERVER_URL=https://s3.example.com  # API 地址
MINIO_BROWSER_REDIRECT_URL=https://console.example.com  # 控制台

# 存储类
MINIO_STORAGE_CLASS_STANDARD=EC:4
MINIO_STORAGE_CLASS_RRS=EC:2

# 压缩
MINIO_COMPRESSION_ENABLED=on
MINIO_COMPRESSION_MIME_TYPES=text/*,application/json

# 监控
MINIO_PROMETHEUS_AUTH_TYPE=public   # Prometheus 认证方式
```

---

# Part 4: Bucket 与对象管理

## 4.1 Bucket 操作

### 4.1.1 Maven 依赖

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.7</version>
</dependency>
```

### 4.1.2 Bucket CRUD

```java
import io.minio.*;
import io.minio.errors.*;
import java.util.*;

public class BucketOperations {

    private final MinioClient minioClient;

    public BucketOperations() {
        this.minioClient = MinioClient.builder()
            .endpoint("http://localhost:9000")
            .credentials("minioadmin", "minioadmin")
            .build();
    }

    // 创建 Bucket
    public void createBucket(String bucketName) throws Exception {
        boolean found = minioClient.bucketExists(
            BucketExistsArgs.builder().bucket(bucketName).build());
        if (!found) {
            minioClient.makeBucket(
                MakeBucketArgs.builder().bucket(bucketName).build());
            System.out.println("Bucket created: " + bucketName);
        } else {
            System.out.println("Bucket already exists: " + bucketName);
        }
    }

    // 列出所有 Bucket
    public void listBuckets() throws Exception {
        List<Bucket> buckets = minioClient.listBuckets();
        for (Bucket bucket : buckets) {
            System.out.printf("Bucket: %-20s Created: %s%n",
                bucket.name(), bucket.creationDate());
        }
    }

    // 删除 Bucket（必须先清空）
    public void deleteBucket(String bucketName) throws Exception {
        minioClient.removeBucket(
            RemoveBucketArgs.builder().bucket(bucketName).build());
        System.out.println("Bucket deleted: " + bucketName);
    }

    // 设置 Bucket 标签
    public void setBucketTags(String bucketName) throws Exception {
        Map<String, String> tags = new HashMap<>();
        tags.put("env", "production");
        tags.put("team", "backend");
        minioClient.setBucketTags(
            SetBucketTagsArgs.builder()
                .bucket(bucketName)
                .tags(tags)
                .build());
    }

    // 开启版本控制
    public void enableVersioning(String bucketName) throws Exception {
        minioClient.setBucketVersioning(
            SetBucketVersioningArgs.builder()
                .bucket(bucketName)
                .config(new VersioningConfiguration(
                    VersioningConfiguration.Status.ENABLED, null))
                .build());
    }
}
```

## 4.2 对象上传（普通上传）

```java
public class ObjectUpload {

    private final MinioClient minioClient;

    // 上传文件（自动选择单次/分片）
    public String uploadFile(String bucket, String objectName,
                            InputStream is, long size,
                            String contentType) throws Exception {
        ObjectWriteResponse response = minioClient.putObject(
            PutObjectArgs.builder()
                .bucket(bucket)
                .object(objectName)
                .stream(is, size, -1)  // -1 表示自动分片(5MB)
                .contentType(contentType)
                .build());
        return response.versionId();
    }

    // 上传本地文件
    public void uploadLocalFile(String bucket, String objectName,
                               String filePath) throws Exception {
        minioClient.uploadObject(
            UploadObjectArgs.builder()
                .bucket(bucket)
                .object(objectName)
                .filename(filePath)
                .build());
    }

    // 上传带自定义元数据
    public void uploadWithMetadata(String bucket, String objectName,
                                  InputStream is, long size) throws Exception {
        Map<String, String> userMetadata = new HashMap<>();
        userMetadata.put("x-amz-meta-author", "zhangsan");
        userMetadata.put("x-amz-meta-category", "image");

        minioClient.putObject(
            PutObjectArgs.builder()
                .bucket(bucket)
                .object(objectName)
                .stream(is, size, -1)
                .userMetadata(userMetadata)
                .contentType("image/jpeg")
                .build());
    }
}
```

## 4.3 分片上传（大文件）

```java
public class MultipartUpload {

    private final MinioClient minioClient;

    /**
     * 手动分片上传，支持断点续传
     * @param bucket     存储桶
     * @param objectName 对象名
     * @param file       本地文件
     * @param partSize   分片大小（建议 10MB ~ 5GB）
     */
    public void multipartUpload(String bucket, String objectName,
                               File file, long partSize) throws Exception {
        // 1. 初始化分片上传，获取 uploadId
        String uploadId = minioClient
            .createMultipartUpload(bucket, null, objectName, null, null)
            .result().uploadId();

        List<Part> parts = new ArrayList<>();
        long fileSize = file.length();
        long offset = 0;
        int partNumber = 1;

        try (RandomAccessFile raf = new RandomAccessFile(file, "r")) {
            while (offset < fileSize) {
                long actualPartSize = Math.min(partSize, fileSize - offset);
                byte[] buffer = new byte[(int) actualPartSize];
                raf.read(buffer);

                // 2. 上传每个分片
                UploadPartResponse partResponse = minioClient.uploadPart(
                    bucket, null, objectName,
                    new ByteArrayInputStream(buffer), actualPartSize,
                    uploadId, partNumber, null, null);

                parts.add(new Part(partNumber, partResponse.etag()));
                offset += actualPartSize;
                partNumber++;
            }
        }

        // 3. 完成分片上传
        minioClient.completeMultipartUpload(
            bucket, null, objectName, uploadId,
            parts.toArray(new Part[0]), null, null);

        System.out.println("Multipart upload completed: " + objectName);
    }

    // 查询未完成的分片上传（用于断点续传）
    public void listIncompleteUploads(String bucket) throws Exception {
        Iterable<Result<Upload>> uploads = minioClient.listIncompleteUploads(
            ListIncompleteUploadsArgs.builder()
                .bucket(bucket)
                .build());
        for (Result<Upload> upload : uploads) {
            Upload u = upload.get();
            System.out.printf("UploadId: %s, Object: %s%n",
                u.uploadId(), u.objectName());
        }
    }
}
```

## 4.4 对象下载

```java
public class ObjectDownload {

    private final MinioClient minioClient;

    // 下载为 InputStream
    public InputStream downloadObject(String bucket,
                                     String objectName) throws Exception {
        return minioClient.getObject(
            GetObjectArgs.builder()
                .bucket(bucket)
                .object(objectName)
                .build());
    }

    // 范围下载（Range Download）
    public InputStream rangeDownload(String bucket, String objectName,
                                    long offset, long length) throws Exception {
        return minioClient.getObject(
            GetObjectArgs.builder()
                .bucket(bucket)
                .object(objectName)
                .offset(offset)
                .length(length)
                .build());
    }

    // 下载到本地文件
    public void downloadToFile(String bucket, String objectName,
                              String localPath) throws Exception {
        minioClient.downloadObject(
            DownloadObjectArgs.builder()
                .bucket(bucket)
                .object(objectName)
                .filename(localPath)
                .build());
    }

    // 获取对象元数据
    public StatObjectResponse getObjectInfo(String bucket,
                                           String objectName) throws Exception {
        return minioClient.statObject(
            StatObjectArgs.builder()
                .bucket(bucket)
                .object(objectName)
                .build());
    }

    // 列出对象（带分页）
    public void listObjects(String bucket, String prefix) throws Exception {
        Iterable<Result<Item>> objects = minioClient.listObjects(
            ListObjectsArgs.builder()
                .bucket(bucket)
                .prefix(prefix)
                .recursive(true)
                .build());
        for (Result<Item> result : objects) {
            Item item = result.get();
            System.out.printf("%-50s %10d %s%n",
                item.objectName(), item.size(), item.lastModified());
        }
    }
}
```

## 4.5 预签名 URL（Presigned URL）

预签名 URL 允许未经认证的用户在指定时间内访问私有对象，非常适合前端直传、临时分享等场景。

```java
public class PresignedUrl {

    private final MinioClient minioClient;

    // 生成下载预签名 URL（GET）
    public String getPresignedDownloadUrl(String bucket, String objectName,
                                         int expireSeconds) throws Exception {
        return minioClient.getPresignedObjectUrl(
            GetPresignedObjectUrlArgs.builder()
                .method(Method.GET)
                .bucket(bucket)
                .object(objectName)
                .expiry(expireSeconds)
                .build());
    }

    // 生成上传预签名 URL（PUT）
    public String getPresignedUploadUrl(String bucket, String objectName,
                                       int expireSeconds) throws Exception {
        return minioClient.getPresignedObjectUrl(
            GetPresignedObjectUrlArgs.builder()
                .method(Method.PUT)
                .bucket(bucket)
                .object(objectName)
                .expiry(expireSeconds)
                .build());
    }

    // 生成 POST 表单预签名（浏览器直传）
    public Map<String, String> getPresignedPostForm(String bucket,
                                                     String objectName) throws Exception {
        PostPolicy policy = new PostPolicy(bucket,
            ZonedDateTime.now().plusHours(1));
        policy.addEqualsCondition("key", objectName);
        // 限制文件大小 1KB ~ 100MB
        policy.addContentLengthRangeCondition(1024, 100 * 1024 * 1024);
        // 限制 Content-Type 为图片
        policy.addStartsWithCondition("Content-Type", "image/");

        return minioClient.getPresignedPostFormData(policy);
    }

    // 生成带自定义响应头的预签名 URL
    public String getPresignedUrlWithHeaders(String bucket,
                                             String objectName) throws Exception {
        Map<String, String> reqParams = new HashMap<>();
        // 强制浏览器下载（而非预览）
        reqParams.put("response-content-disposition",
            "attachment; filename=downloaded.pdf");

        return minioClient.getPresignedObjectUrl(
            GetPresignedObjectUrlArgs.builder()
                .method(Method.GET)
                .bucket(bucket)
                .object(objectName)
                .expiry(3600)
                .extraQueryParams(reqParams)
                .build());
    }
}
```

## 4.6 对象生命周期管理（ILM）

```java
public class LifecycleManagement {

    private final MinioClient minioClient;

    public void setLifecycleRules(String bucket) throws Exception {
        List<LifecycleRule> rules = new ArrayList<>();

        // 规则1: 30天后过期（针对 logs/ 前缀）
        rules.add(new LifecycleRule(
            Status.ENABLED,
            null,
            new Expiration((ZonedDateTime) null, 30, null),
            new RuleFilter("logs/"),
            "rule-expire-logs",
            null, null, null
        ));

        // 规则2: 90天后永久删除（针对 temp/ 前缀）
        rules.add(new LifecycleRule(
            Status.ENABLED,
            null,
            new Expiration((ZonedDateTime) null, 90, null),
            new RuleFilter("temp/"),
            "rule-delete-temp",
            null, null, null
        ));

        minioClient.setBucketLifecycle(
            SetBucketLifecycleArgs.builder()
                .bucket(bucket)
                .config(new LifecycleConfiguration(rules))
                .build());
    }

    public void getLifecycleRules(String bucket) throws Exception {
        LifecycleConfiguration config = minioClient.getBucketLifecycle(
            GetBucketLifecycleArgs.builder().bucket(bucket).build());
        config.rules().forEach(rule ->
            System.out.printf("Rule: %s, Status: %s%n",
                rule.id(), rule.status()));
    }
}
```

## 4.7 对象复制与移动

```java
public class ObjectCopyMove {

    private final MinioClient minioClient;

    // 复制对象（同桶或跨桶）
    public void copyObject(String srcBucket, String srcObject,
                          String dstBucket, String dstObject) throws Exception {
        minioClient.copyObject(
            CopyObjectArgs.builder()
                .bucket(dstBucket)
                .object(dstObject)
                .source(CopySource.builder()
                    .bucket(srcBucket)
                    .object(srcObject)
                    .build())
                .build());
    }

    // 移动对象 = 复制 + 删除源
    public void moveObject(String srcBucket, String srcObject,
                          String dstBucket, String dstObject) throws Exception {
        copyObject(srcBucket, srcObject, dstBucket, dstObject);
        minioClient.removeObject(
            RemoveObjectArgs.builder()
                .bucket(srcBucket)
                .object(srcObject)
                .build());
    }

    // 批量删除对象
    public void batchDelete(String bucket,
                           List<String> objectNames) throws Exception {
        List<DeleteObject> objects = objectNames.stream()
            .map(DeleteObject::new)
            .collect(Collectors.toList());

        Iterable<Result<DeleteError>> results = minioClient.removeObjects(
            RemoveObjectsArgs.builder()
                .bucket(bucket)
                .objects(objects)
                .build());

        for (Result<DeleteError> result : results) {
            DeleteError error = result.get();
            System.err.printf("Failed to delete %s: %s%n",
                error.objectName(), error.message());
        }
    }
}
```

---

# Part 5: 权限与安全

## 5.1 Bucket Policy（存储桶策略）

MinIO 的存储桶策略基于 AWS IAM Policy 语法，使用 JSON 格式定义。

### 5.1.1 公共读策略

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": ["*"]},
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::public-bucket/*"]
    }
  ]
}
```

### 5.1.2 特定 IP 限制策略

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": ["*"]},
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": ["arn:aws:s3:::internal-bucket/*"],
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": ["192.168.1.0/24", "10.0.0.0/8"]
        }
      }
    }
  ]
}
```

### 5.1.3 全部 S3 Action 说明

| Action | 描述 |
|--------|------|
| s3:GetObject | 下载对象 |
| s3:PutObject | 上传对象 |
| s3:DeleteObject | 删除对象 |
| s3:ListBucket | 列出桶内对象 |
| s3:GetBucketLocation | 获取桶区域 |
| s3:GetObjectVersion | 获取对象版本 |
| s3:DeleteObjectVersion | 删除对象版本 |
| s3:GetBucketVersioning | 获取版本控制状态 |
| s3:PutBucketVersioning | 设置版本控制 |
| s3:GetLifecycleConfiguration | 获取生命周期 |
| s3:PutLifecycleConfiguration | 设置生命周期 |

### 5.1.4 Java 设置存储桶策略

```java
public void setBucketPolicy(String bucket, String policyJson) throws Exception {
    minioClient.setBucketPolicy(
        SetBucketPolicyArgs.builder()
            .bucket(bucket)
            .config(policyJson)
            .build());
}

public String getBucketPolicy(String bucket) throws Exception {
    return minioClient.getBucketPolicy(
        GetBucketPolicyArgs.builder().bucket(bucket).build());
}
```

## 5.2 IAM 用户与策略管理

```bash
# 创建用户
mc admin user add myminio alice alicepassword123

# 列出用户
mc admin user list myminio

# 禁用/启用用户
mc admin user disable myminio alice
mc admin user enable myminio alice

# 创建策略文件 my-policy.json
mc admin policy create myminio my-policy my-policy.json

# 为用户附加策略
mc admin policy attach myminio my-policy --user alice

# 创建服务账户（Access Key/Secret Key 对）
mc admin accesskey create myminio alice

# 创建用户组
mc admin group add myminio backend-team alice bob
mc admin policy attach myminio readwrite --group backend-team
```

## 5.3 内置策略说明

| 策略名称 | 权限说明 |
|---------|----------|
| `readwrite` | 对所有 Bucket 有完整读写权限 |
| `readonly` | 对所有 Bucket 只读 |
| `writeonly` | 对所有 Bucket 只写 |
| `diagnostics` | 访问诊断和监控端点 |
| `consoleAdmin` | 控制台完整管理权限 |

## 5.4 TLS/HTTPS 配置

```bash
# 方式1: 使用自签名证书（测试用）
openssl req -x509 -nodes -days 730 -newkey rsa:2048 \
  -keyout private.key -out public.crt \
  -subj "/CN=minio.local"

mkdir -p ~/.minio/certs
cp public.crt ~/.minio/certs/
cp private.key ~/.minio/certs/

# 方式2: 使用 Let's Encrypt
certbot certonly --standalone -d s3.example.com
ln -s /etc/letsencrypt/live/s3.example.com/fullchain.pem \
      ~/.minio/certs/public.crt
ln -s /etc/letsencrypt/live/s3.example.com/privkey.pem \
      ~/.minio/certs/private.key

# 方式3: Docker 挂载证书目录
docker run -v /path/to/certs:/root/.minio/certs \
  quay.io/minio/minio server /data
```

## 5.5 服务端加密（SSE）

```
+-------------------+-----------------------------+--------------------+
| 加密模式          | 密钥管理                    | 备注               |
+-------------------+-----------------------------+--------------------+
| SSE-C             | 客户端提供，MinIO 不存储    | 客户端负责密钥保管 |
| SSE-KMS           | 外部 KMS（Vault/KES）       | 推荐生产环境使用   |
| SSE-S3 (auto)     | MinIO 自动管理              | 透明加密           |
+-------------------+-----------------------------+--------------------+
```

```java
// SSE-C 加密上传
byte[] myKey = "my-32-byte-encryption-key-123456".getBytes();
ServerSideEncryptionCustomerKey ssec =
    new ServerSideEncryptionCustomerKey(myKey);

minioClient.putObject(
    PutObjectArgs.builder()
        .bucket("mybucket")
        .object("encrypted-file.txt")
        .stream(is, size, -1)
        .sse(ssec)
        .build());

// SSE-C 解密下载（必须提供相同密钥）
InputStream result = minioClient.getObject(
    GetObjectArgs.builder()
        .bucket("mybucket")
        .object("encrypted-file.txt")
        .ssec(ssec)
        .build());
```

## 5.6 静态网站托管

```bash
# 配置 Bucket 为静态网站模式
mc website set myminio/my-website \
    --index-document index.html \
    --error-document error.html

# 或通过 Java SDK
```

```java
public void enableStaticWebsite(String bucket) throws Exception {
    String config = "{\"IndexDocument\":{\"Suffix\":\"index.html\"}," +
        "\"ErrorDocument\":{\"Key\":\"error.html\"}}";
    minioClient.setBucketWebsite(
        SetBucketWebsiteArgs.builder()
            .bucket(bucket)
            .websiteConfiguration(
                WebsiteConfiguration.parseJson(config))
            .build());
}
```

---

# Part 6: 通知与事件

## 6.1 事件通知概述

MinIO 支持当对象发生变化时向外部系统推送事件，支持的通知目标系统：

```
+--------------------------------------------+
|           MinIO 事件通知架构               |
+--------------------------------------------+
|                                            |
|  对象操作  -->  事件引擎                   |
|  (PUT/GET/DELETE)   |                      |
|                     v                      |
|            +------------------+            |
|            |   事件队列        |            |
|            |  (持久化/重试)    |            |
|            +------------------+            |
|                     |                      |
|    +----------------+----------------+     |
|    |                |                |     |
|  Kafka           Webhook          Redis    |
|  AMQP            MySQL            NSQ      |
|  MQTT            PostgreSQL       ES       |
+--------------------------------------------+
```

## 6.2 支持的事件类型

| 事件名称 | 触发时机 |
|---------|----------|
| s3:ObjectCreated:* | 任何创建操作 |
| s3:ObjectCreated:Put | PUT 上传 |
| s3:ObjectCreated:Post | POST 上传 |
| s3:ObjectCreated:Copy | 复制操作 |
| s3:ObjectCreated:CompleteMultipartUpload | 分片上传完成 |
| s3:ObjectRemoved:* | 任何删除操作 |
| s3:ObjectRemoved:Delete | DELETE 删除 |
| s3:ObjectRemoved:DeleteMarkerCreated | 创建删除标记 |
| s3:ObjectAccessed:Get | GET 下载 |
| s3:ObjectAccessed:Head | HEAD 请求 |

## 6.3 Kafka 事件通知配置

```bash
# 配置 Kafka 通知目标（notify_kafka:ID 中的 ID 为自定义标识）
mc admin config set myminio notify_kafka:1 \
    brokers="kafka1:9092,kafka2:9092" \
    topic="minio-events" \
    tls=off \
    tls_skip_verify=off \
    queue_limit=10000 \
    queue_dir=/tmp/kafka-events

# 重启 MinIO 使配置生效
mc admin service restart myminio

# 为指定 Bucket 添加事件监听
mc event add myminio/mybucket arn:minio:sqs::1:kafka \
    --event put,delete,get \
    --prefix uploads/ \
    --suffix .jpg,.png

# 查看事件列表
mc event list myminio/mybucket

# 删除事件绑定
mc event remove myminio/mybucket arn:minio:sqs::1:kafka
```

## 6.4 Webhook 事件通知配置

```bash
# 配置 Webhook 目标
mc admin config set myminio notify_webhook:mywebhook \
    endpoint="https://api.example.com/minio/events" \
    auth_token="Bearer your-token-here" \
    queue_limit=10000 \
    queue_dir=/tmp/webhook-events

mc admin service restart myminio

# 绑定 Bucket 事件
mc event add myminio/uploads arn:minio:sqs::mywebhook:webhook \
    --event put
```

## 6.5 事件消息格式（JSON）

```json
{
  "EventName": "s3:ObjectCreated:Put",
  "Key": "mybucket/uploads/photo.jpg",
  "Records": [
    {
      "eventVersion": "2.0",
      "eventSource": "minio:s3",
      "awsRegion": "",
      "eventTime": "2024-01-15T10:30:00.000Z",
      "eventName": "s3:ObjectCreated:Put",
      "userIdentity": {"principalId": "alice"},
      "requestParameters": {
        "principalId": "alice",
        "sourceIPAddress": "192.168.1.100"
      },
      "s3": {
        "s3SchemaVersion": "1.0",
        "bucket": {
          "name": "mybucket",
          "arn": "arn:aws:s3:::mybucket"
        },
        "object": {
          "key": "uploads/photo.jpg",
          "size": 1024576,
          "eTag": "d41d8cd98f00b204e9800998ecf8427e",
          "contentType": "image/jpeg",
          "userMetadata": {
            "x-amz-meta-author": "zhangsan"
          }
        }
      }
    }
  ]
}
```

## 6.6 Spring Boot 消费 MinIO Kafka 事件

```java
@Component
@Slf4j
public class MinioEventConsumer {

    private final ObjectMapper objectMapper;
    private final ImageProcessingService imageService;

    public MinioEventConsumer(ObjectMapper objectMapper,
                             ImageProcessingService imageService) {
        this.objectMapper = objectMapper;
        this.imageService = imageService;
    }

    @KafkaListener(topics = "minio-events", groupId = "minio-event-group")
    public void consume(ConsumerRecord<String, String> record) {
        try {
            MinioEvent event = objectMapper.readValue(
                record.value(), MinioEvent.class);

            for (MinioEventRecord r : event.getRecords()) {
                handleEvent(r);
            }
        } catch (Exception e) {
            log.error("Failed to process MinIO event: {}", record.value(), e);
        }
    }

    private void handleEvent(MinioEventRecord record) {
        String eventName = record.getEventName();
        String bucket = record.getS3().getBucket().getName();
        String objectKey = record.getS3().getObject().getKey();

        log.info("MinIO Event: {} - {}/{}", eventName, bucket, objectKey);

        switch (eventName) {
            case "s3:ObjectCreated:Put" -> {
                // 图片上传时自动生成缩略图
                if (objectKey.matches(".*\\.(jpg|png|jpeg|gif)$")) {
                    imageService.generateThumbnail(bucket, objectKey);
                }
            }
            case "s3:ObjectRemoved:Delete" -> {
                // 删除时同步删除缩略图
                imageService.deleteThumbnail(bucket, objectKey);
            }
        }
    }
}

@Data
public class MinioEvent {
    @JsonProperty("EventName")
    private String eventName;
    @JsonProperty("Key")
    private String key;
    @JsonProperty("Records")
    private List<MinioEventRecord> records;
}

@Data
public class MinioEventRecord {
    private String eventVersion;
    private String eventName;
    private String eventTime;
    private S3Info s3;
}

@Data
public class S3Info {
    private BucketInfo bucket;
    private ObjectInfo object;
}

@Data
public class BucketInfo {
    private String name;
    private String arn;
}

@Data
public class ObjectInfo {
    private String key;
    private long size;
    private String eTag;
    private String contentType;
}
```

---

# Part 7: Spring Boot 集成

## 7.1 依赖与配置

### 7.1.1 Maven 依赖

```xml
<dependencies>
    <!-- MinIO Java SDK -->
    <dependency>
        <groupId>io.minio</groupId>
        <artifactId>minio</artifactId>
        <version>8.5.7</version>
    </dependency>

    <!-- OkHttp (MinIO SDK 依赖) -->
    <dependency>
        <groupId>com.squareup.okhttp3</groupId>
        <artifactId>okhttp</artifactId>
        <version>4.12.0</version>
    </dependency>

    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

### 7.1.2 application.yml 配置

```yaml
minio:
  endpoint: http://localhost:9000
  access-key: minioadmin
  secret-key: minioadmin123
  bucket-name: default-bucket
  # 文件访问域名（CDN 或反代）
  domain: https://cdn.example.com

spring:
  servlet:
    multipart:
      max-file-size: 500MB
      max-request-size: 500MB
```

### 7.1.3 MinIO 配置类

```java
@Configuration
@ConfigurationProperties(prefix = "minio")
@Data
public class MinioProperties {
    private String endpoint;
    private String accessKey;
    private String secretKey;
    private String bucketName;
    private String domain;
}

@Configuration
public class MinioConfig {

    @Bean
    public MinioClient minioClient(MinioProperties props) {
        return MinioClient.builder()
            .endpoint(props.getEndpoint())
            .credentials(props.getAccessKey(), props.getSecretKey())
            .build();
    }
}
```

## 7.2 MinIOTemplate 工具类

```java
@Component
@Slf4j
public class MinioTemplate {

    private final MinioClient minioClient;
    private final MinioProperties minioProperties;

    public MinioTemplate(MinioClient minioClient,
                        MinioProperties minioProperties) {
        this.minioClient = minioClient;
        this.minioProperties = minioProperties;
    }

    /**
     * 确保 Bucket 存在，不存在则创建
     */
    public void ensureBucketExists(String bucketName) {
        try {
            boolean exists = minioClient.bucketExists(
                BucketExistsArgs.builder().bucket(bucketName).build());
            if (!exists) {
                minioClient.makeBucket(
                    MakeBucketArgs.builder().bucket(bucketName).build());
                log.info("Created bucket: {}", bucketName);
            }
        } catch (Exception e) {
            throw new RuntimeException("Failed to ensure bucket: " + bucketName, e);
        }
    }

    /**
     * 上传文件（返回访问 URL）
     *
     * @param multipartFile Spring 上传文件
     * @param folder        存储目录，如 "avatar" or "docs/2024"
     * @return 文件访问 URL
     */
    public String uploadFile(MultipartFile multipartFile, String folder) {
        try {
            String bucket = minioProperties.getBucketName();
            ensureBucketExists(bucket);

            // 生成唯一文件名（保留原始扩展名）
            String originalFilename = multipartFile.getOriginalFilename();
            String ext = originalFilename != null ?
                originalFilename.substring(originalFilename.lastIndexOf(".")) : "";
            String objectName = folder + "/" + UUID.randomUUID() + ext;

            minioClient.putObject(
                PutObjectArgs.builder()
                    .bucket(bucket)
                    .object(objectName)
                    .stream(multipartFile.getInputStream(),
                        multipartFile.getSize(), -1)
                    .contentType(multipartFile.getContentType())
                    .build());

            log.info("Uploaded: {}/{}", bucket, objectName);
            return buildUrl(objectName);
        } catch (Exception e) {
            throw new RuntimeException("Upload failed", e);
        }
    }

    /**
     * 获取文件预览 URL（永久公开）
     */
    public String getPublicUrl(String objectName) {
        return buildUrl(objectName);
    }

    /**
     * 获取临时访问 URL（预签名，默认1小时）
     */
    public String getPresignedUrl(String objectName, int expireSeconds) {
        try {
            return minioClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                    .method(Method.GET)
                    .bucket(minioProperties.getBucketName())
                    .object(objectName)
                    .expiry(expireSeconds)
                    .build());
        } catch (Exception e) {
            throw new RuntimeException("Failed to get presigned URL", e);
        }
    }

    /**
     * 获取前端直传预签名 URL（PUT 方法）
     */
    public String getPresignedPutUrl(String objectName, int expireSeconds) {
        try {
            return minioClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                    .method(Method.PUT)
                    .bucket(minioProperties.getBucketName())
                    .object(objectName)
                    .expiry(expireSeconds)
                    .build());
        } catch (Exception e) {
            throw new RuntimeException("Failed to get PUT presigned URL", e);
        }
    }

    /**
     * 下载文件（返回 InputStream）
     */
    public InputStream downloadFile(String objectName) {
        try {
            return minioClient.getObject(
                GetObjectArgs.builder()
                    .bucket(minioProperties.getBucketName())
                    .object(objectName)
                    .build());
        } catch (Exception e) {
            throw new RuntimeException("Download failed: " + objectName, e);
        }
    }

    /**
     * 删除文件
     */
    public void deleteFile(String objectName) {
        try {
            minioClient.removeObject(
                RemoveObjectArgs.builder()
                    .bucket(minioProperties.getBucketName())
                    .object(objectName)
                    .build());
            log.info("Deleted: {}", objectName);
        } catch (Exception e) {
            throw new RuntimeException("Delete failed: " + objectName, e);
        }
    }

    /**
     * 获取对象元数据
     */
    public StatObjectResponse getObjectStat(String objectName) {
        try {
            return minioClient.statObject(
                StatObjectArgs.builder()
                    .bucket(minioProperties.getBucketName())
                    .object(objectName)
                    .build());
        } catch (Exception e) {
            throw new RuntimeException("Stat failed: " + objectName, e);
        }
    }

    /**
     * 检查对象是否存在
     */
    public boolean objectExists(String objectName) {
        try {
            minioClient.statObject(
                StatObjectArgs.builder()
                    .bucket(minioProperties.getBucketName())
                    .object(objectName)
                    .build());
            return true;
        } catch (ErrorResponseException e) {
            if (e.errorResponse().code().equals("NoSuchKey")) {
                return false;
            }
            throw new RuntimeException("Check failed: " + objectName, e);
        } catch (Exception e) {
            throw new RuntimeException("Check failed: " + objectName, e);
        }
    }

    private String buildUrl(String objectName) {
        String domain = minioProperties.getDomain();
        if (domain != null && !domain.isEmpty()) {
            return domain + "/" + objectName;
        }
        return minioProperties.getEndpoint() + "/" +
            minioProperties.getBucketName() + "/" + objectName;
    }
}
```

## 7.3 文件上传 REST 接口

```java
@RestController
@RequestMapping("/api/files")
@Slf4j
public class FileController {

    private final MinioTemplate minioTemplate;

    public FileController(MinioTemplate minioTemplate) {
        this.minioTemplate = minioTemplate;
    }

    /**
     * 单文件上传
     */
    @PostMapping("/upload")
    public Result<FileUploadVO> upload(
            @RequestParam("file") MultipartFile file,
            @RequestParam(value = "folder", defaultValue = "default") String folder) {
        if (file.isEmpty()) {
            return Result.fail("文件不能为空");
        }

        String url = minioTemplate.uploadFile(file, folder);

        FileUploadVO vo = FileUploadVO.builder()
            .fileName(file.getOriginalFilename())
            .fileSize(file.getSize())
            .url(url)
            .build();

        return Result.ok(vo);
    }

    /**
     * 批量上传
     */
    @PostMapping("/batch-upload")
    public Result<List<FileUploadVO>> batchUpload(
            @RequestParam("files") List<MultipartFile> files,
            @RequestParam(value = "folder", defaultValue = "default") String folder) {
        List<FileUploadVO> results = files.stream()
            .filter(f -> !f.isEmpty())
            .map(f -> {
                String url = minioTemplate.uploadFile(f, folder);
                return FileUploadVO.builder()
                    .fileName(f.getOriginalFilename())
                    .fileSize(f.getSize())
                    .url(url)
                    .build();
            })
            .collect(Collectors.toList());

        return Result.ok(results);
    }

    /**
     * 获取预签名上传 URL（前端直传）
     */
    @GetMapping("/presigned-upload-url")
    public Result<PresignedUrlVO> getPresignedUploadUrl(
            @RequestParam String filename,
            @RequestParam(value = "folder", defaultValue = "default") String folder) {
        String objectName = folder + "/" + UUID.randomUUID() +
            filename.substring(filename.lastIndexOf("."));
        String presignedUrl = minioTemplate.getPresignedPutUrl(objectName, 3600);

        PresignedUrlVO vo = PresignedUrlVO.builder()
            .objectName(objectName)
            .uploadUrl(presignedUrl)
            .expireSeconds(3600)
            .build();

        return Result.ok(vo);
    }

    /**
     * 文件下载
     */
    @GetMapping("/download")
    public void download(@RequestParam String objectName,
                        HttpServletResponse response) throws IOException {
        StatObjectResponse stat = minioTemplate.getObjectStat(objectName);

        response.setContentType(stat.contentType());
        response.setHeader("Content-Disposition",
            "attachment; filename=" +
            URLEncoder.encode(objectName, "UTF-8").replace("+", "%20"));
        response.setContentLengthLong(stat.size());

        try (InputStream is = minioTemplate.downloadFile(objectName);
             OutputStream os = response.getOutputStream()) {
            StreamUtils.copy(is, os);
        }
    }

    /**
     * 删除文件
     */
    @DeleteMapping
    public Result<Void> delete(@RequestParam String objectName) {
        minioTemplate.deleteFile(objectName);
        return Result.ok();
    }
}

@Data
@Builder
public class FileUploadVO {
    private String fileName;
    private long fileSize;
    private String url;
}

@Data
@Builder
public class PresignedUrlVO {
    private String objectName;
    private String uploadUrl;
    private int expireSeconds;
}
```

## 7.4 断点续传实现

断点续传的核心是：记录已上传的 Part，重试时跳过已完成的部分。

```java
@Service
@Slf4j
public class ResumableUploadService {

    private final MinioClient minioClient;
    private final MinioProperties props;
    // 存储 uploadId 的 Map（生产环境应存 Redis）
    private final Map<String, String> uploadIdCache =
        new ConcurrentHashMap<>();

    private static final long PART_SIZE = 10 * 1024 * 1024; // 10MB

    /**
     * 初始化分片上传，返回 uploadId
     */
    public String initMultipartUpload(String objectName,
                                      String contentType) throws Exception {
        Map<String, String> headers = new HashMap<>();
        headers.put("Content-Type", contentType);

        String uploadId = minioClient
            .createMultipartUpload(props.getBucketName(), null, objectName,
                Multimap.of("Content-Type", contentType), null)
            .result().uploadId();

        uploadIdCache.put(objectName, uploadId);
        log.info("Init multipart upload: objectName={}, uploadId={}", objectName, uploadId);
        return uploadId;
    }

    /**
     * 上传单个分片
     */
    public String uploadPart(String objectName, String uploadId,
                            int partNumber, byte[] data) throws Exception {
        UploadPartResponse response = minioClient.uploadPart(
            props.getBucketName(), null, objectName,
            new ByteArrayInputStream(data), data.length,
            uploadId, partNumber, null, null);

        log.info("Uploaded part {}/{}: etag={}", partNumber, objectName, response.etag());
        return response.etag();
    }

    /**
     * 完成分片上传
     */
    public String completeMultipartUpload(String objectName, String uploadId,
                                          List<Part> parts) throws Exception {
        minioClient.completeMultipartUpload(
            props.getBucketName(), null, objectName, uploadId,
            parts.toArray(new Part[0]), null, null);

        uploadIdCache.remove(objectName);
        log.info("Completed multipart upload: {}", objectName);
        return props.getEndpoint() + "/" + props.getBucketName() + "/" + objectName;
    }

    /**
     * 取消分片上传（释放临时存储空间）
     */
    public void abortMultipartUpload(String objectName) throws Exception {
        String uploadId = uploadIdCache.get(objectName);
        if (uploadId != null) {
            minioClient.abortMultipartUpload(
                props.getBucketName(), null, objectName, uploadId, null, null);
            uploadIdCache.remove(objectName);
            log.info("Aborted multipart upload: {}", objectName);
        }
    }
}
```

### 7.4.1 断点续传 REST 接口

```java
@RestController
@RequestMapping("/api/upload/multipart")
public class MultipartUploadController {

    private final ResumableUploadService uploadService;

    // Step 1: 初始化
    @PostMapping("/init")
    public Result<String> init(@RequestParam String filename,
                              @RequestParam String contentType) throws Exception {
        String objectName = "uploads/" + UUID.randomUUID() + "_" + filename;
        String uploadId = uploadService.initMultipartUpload(objectName, contentType);
        // 返回 uploadId 给前端，前端需要持久化
        return Result.ok(uploadId);
    }

    // Step 2: 上传分片（前端循环调用）
    @PostMapping("/part")
    public Result<String> uploadPart(
            @RequestParam String objectName,
            @RequestParam String uploadId,
            @RequestParam int partNumber,
            @RequestParam("data") MultipartFile data) throws Exception {
        String etag = uploadService.uploadPart(objectName, uploadId,
            partNumber, data.getBytes());
        return Result.ok(etag);
    }

    // Step 3: 完成上传
    @PostMapping("/complete")
    public Result<String> complete(
            @RequestParam String objectName,
            @RequestParam String uploadId,
            @RequestBody List<PartInfo> parts) throws Exception {
        List<Part> partList = parts.stream()
            .map(p -> new Part(p.getPartNumber(), p.getEtag()))
            .collect(Collectors.toList());
        String url = uploadService.completeMultipartUpload(
            objectName, uploadId, partList);
        return Result.ok(url);
    }

    // 取消上传
    @DeleteMapping("/abort")
    public Result<Void> abort(@RequestParam String objectName) throws Exception {
        uploadService.abortMultipartUpload(objectName);
        return Result.ok();
    }
}
```

## 7.5 图片处理（缩略图生成）

```java
@Service
@Slf4j
public class ImageProcessingService {

    private final MinioTemplate minioTemplate;

    /**
     * 生成缩略图并上传到 MinIO
     * 依赖: thumbnailator
     */
    public void generateThumbnail(String bucket, String objectKey) {
        try {
            // 1. 从 MinIO 下载原图
            InputStream original = minioTemplate.downloadFile(objectKey);

            // 2. 生成缩略图（200x200，保持比例）
            ByteArrayOutputStream thumbOut = new ByteArrayOutputStream();
            Thumbnails.of(original)
                .size(200, 200)
                .keepAspectRatio(true)
                .outputFormat("jpg")
                .toOutputStream(thumbOut);

            // 3. 上传缩略图（存储在 thumbnail/ 目录下）
            String thumbKey = "thumbnail/" +
                objectKey.replaceFirst("uploads/", "");

            minioClient.putObject(
                PutObjectArgs.builder()
                    .bucket(bucket)
                    .object(thumbKey)
                    .stream(new ByteArrayInputStream(thumbOut.toByteArray()),
                        thumbOut.size(), -1)
                    .contentType("image/jpeg")
                    .build());

            log.info("Generated thumbnail: {}", thumbKey);
        } catch (Exception e) {
            log.error("Failed to generate thumbnail for: {}", objectKey, e);
        }
    }
}
```

---

# Part 8: 实战案例

## 8.1 案例一：用户头像服务

```
需求:
  - 用户上传头像（限制格式：jpg/png/gif，大小 < 5MB）
  - 自动生成多尺寸缩略图（64x64、128x128、256x256）
  - 旧头像自动删除，节省存储
  - 头像 CDN 加速访问

架构:
  前端 --> POST /api/avatar --> AvatarService --> MinIO
       --> Kafka 事件 --> ThumbnailWorker --> MinIO (缩略图)
```

```java
@Service
@Slf4j
public class AvatarService {

    private final MinioTemplate minioTemplate;
    private final UserRepository userRepository;

    private static final Set<String> ALLOWED_TYPES =
        Set.of("image/jpeg", "image/png", "image/gif");
    private static final long MAX_SIZE = 5 * 1024 * 1024; // 5MB
    private static final int[] THUMB_SIZES = {64, 128, 256};

    public AvatarUploadResult uploadAvatar(Long userId,
                                           MultipartFile file) {
        // 1. 校验文件
        if (!ALLOWED_TYPES.contains(file.getContentType())) {
            throw new BusinessException("仅支持 jpg/png/gif 格式");
        }
        if (file.getSize() > MAX_SIZE) {
            throw new BusinessException("文件大小不能超过 5MB");
        }

        // 2. 删除旧头像
        User user = userRepository.findById(userId).orElseThrow();
        if (user.getAvatarKey() != null) {
            minioTemplate.deleteFile(user.getAvatarKey());
            for (int size : THUMB_SIZES) {
                minioTemplate.deleteFile(
                    "thumbnail/" + size + "/" +
                    user.getAvatarKey().replace("avatar/", ""));
            }
        }

        // 3. 上传新头像
        String url = minioTemplate.uploadFile(file, "avatar");
        String objectKey = extractObjectKey(url);

        // 4. 生成多尺寸缩略图
        Map<Integer, String> thumbUrls = new HashMap<>();
        for (int size : THUMB_SIZES) {
            String thumbUrl = generateAndUploadThumbnail(file, objectKey, size);
            thumbUrls.put(size, thumbUrl);
        }

        // 5. 更新用户头像记录
        user.setAvatarKey(objectKey);
        user.setAvatarUrl(url);
        userRepository.save(user);

        return AvatarUploadResult.builder()
            .originalUrl(url)
            .thumbnails(thumbUrls)
            .build();
    }

    private String generateAndUploadThumbnail(MultipartFile file,
                                              String objectKey, int size) {
        try {
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            Thumbnails.of(file.getInputStream())
                .size(size, size)
                .keepAspectRatio(true)
                .outputFormat("jpg")
                .toOutputStream(out);

            String thumbKey = "thumbnail/" + size + "/" +
                objectKey.replace("avatar/", "");

            byte[] data = out.toByteArray();
            minioClient.putObject(
                PutObjectArgs.builder()
                    .bucket(minioProperties.getBucketName())
                    .object(thumbKey)
                    .stream(new ByteArrayInputStream(data), data.length, -1)
                    .contentType("image/jpeg")
                    .build());

            return minioTemplate.getPublicUrl(thumbKey);
        } catch (Exception e) {
            log.error("Failed to generate thumbnail: size={}, key={}", size, objectKey, e);
            return null;
        }
    }
}
```

## 8.2 案例二：大文件分片上传（前端直传）

```
架构:
  前端 --> 后端(/api/presign) --> 获取预签名分片 URL
  前端 --> 直接 PUT 到 MinIO（绕过后端，减少带宽压力）
  前端 --> 后端(/api/complete) --> 通知合并
```

```java
@RestController
@RequestMapping("/api/direct-upload")
public class DirectUploadController {

    private final MinioClient minioClient;
    private final MinioProperties props;

    /**
     * 获取所有分片的预签名 URL
     */
    @PostMapping("/presign")
    public Result<DirectUploadInitVO> presign(
            @RequestBody DirectUploadRequest req) throws Exception {
        String objectName = "uploads/" + UUID.randomUUID() + "_" + req.getFilename();

        // 初始化分片上传
        String uploadId = minioClient
            .createMultipartUpload(props.getBucketName(), null, objectName,
                null, null)
            .result().uploadId();

        // 计算总分片数
        long partSize = 10 * 1024 * 1024; // 10MB
        int partCount = (int) Math.ceil((double) req.getFileSize() / partSize);

        // 为每个分片生成预签名 URL
        List<PartPresignVO> partUrls = new ArrayList<>();
        for (int i = 1; i <= partCount; i++) {
            // 构造预签名 URL（包含 partNumber 和 uploadId 参数）
            Map<String, String> extraParams = new HashMap<>();
            extraParams.put("partNumber", String.valueOf(i));
            extraParams.put("uploadId", uploadId);

            String presignedUrl = minioClient.getPresignedObjectUrl(
                GetPresignedObjectUrlArgs.builder()
                    .method(Method.PUT)
                    .bucket(props.getBucketName())
                    .object(objectName)
                    .expiry(3600 * 24) // 24小时内有效
                    .extraQueryParams(extraParams)
                    .build());

            partUrls.add(PartPresignVO.builder()
                .partNumber(i)
                .url(presignedUrl)
                .build());
        }

        return Result.ok(DirectUploadInitVO.builder()
            .objectName(objectName)
            .uploadId(uploadId)
            .partUrls(partUrls)
            .build());
    }

    /**
     * 完成分片上传
     */
    @PostMapping("/complete")
    public Result<String> complete(
            @RequestBody DirectUploadCompleteRequest req) throws Exception {
        List<Part> parts = req.getParts().stream()
            .map(p -> new Part(p.getPartNumber(), p.getEtag()))
            .collect(Collectors.toList());

        minioClient.completeMultipartUpload(
            props.getBucketName(), null, req.getObjectName(),
            req.getUploadId(), parts.toArray(new Part[0]), null, null);

        String accessUrl = props.getDomain() + "/" + req.getObjectName();
        return Result.ok(accessUrl);
    }
}
```

## 8.3 案例三：私有文件授权访问

```java
@RestController
@RequestMapping("/api/private-files")
public class PrivateFileController {

    private final MinioTemplate minioTemplate;
    private final FilePermissionService permissionService;

    /**
     * 获取私有文件的临时访问 URL
     * 需要验证当前用户是否有权限访问该文件
     */
    @GetMapping("/access-url")
    public Result<String> getAccessUrl(@RequestParam String fileId,
                                       @AuthenticationPrincipal UserDetails user) {
        // 1. 校验权限
        if (!permissionService.canAccess(user.getUsername(), fileId)) {
            throw new AccessDeniedException("无权访问此文件");
        }

        // 2. 从数据库获取对象Key
        String objectKey = permissionService.getObjectKey(fileId);

        // 3. 生成1小时有效的预签名 URL
        String presignedUrl = minioTemplate.getPresignedUrl(objectKey, 3600);

        return Result.ok(presignedUrl);
    }
}
```

## 8.4 案例四：版本管理与回滚

```java
@Service
public class DocumentVersionService {

    private final MinioClient minioClient;
    private final MinioProperties props;

    // 列出文档的所有历史版本
    public List<DocumentVersion> listVersions(String docKey) throws Exception {
        List<DocumentVersion> versions = new ArrayList<>();

        Iterable<Result<Item>> items = minioClient.listObjects(
            ListObjectsArgs.builder()
                .bucket(props.getBucketName())
                .prefix(docKey)
                .includeVersions(true)
                .build());

        for (Result<Item> result : items) {
            Item item = result.get();
            if (!item.isDeleteMarker()) {
                versions.add(DocumentVersion.builder()
                    .versionId(item.versionId())
                    .size(item.size())
                    .lastModified(item.lastModified())
                    .isLatest(item.isLatest())
                    .build());
            }
        }

        return versions;
    }

    // 回滚到指定版本（复制指定版本内容为最新版本）
    public void rollbackToVersion(String docKey, String versionId) throws Exception {
        minioClient.copyObject(
            CopyObjectArgs.builder()
                .bucket(props.getBucketName())
                .object(docKey)
                .source(CopySource.builder()
                    .bucket(props.getBucketName())
                    .object(docKey)
                    .versionId(versionId)
                    .build())
                .build());
    }

    // 删除指定版本
    public void deleteVersion(String docKey, String versionId) throws Exception {
        minioClient.removeObject(
            RemoveObjectArgs.builder()
                .bucket(props.getBucketName())
                .object(docKey)
                .versionId(versionId)
                .build());
    }
}
```

## 8.5 案例五：文件秒传（MD5 查重）

```java
@Service
@Slf4j
public class FastUploadService {

    private final MinioTemplate minioTemplate;
    private final FileRecordRepository fileRecordRepo;

    /**
     * 秒传：如果相同 MD5 文件已存在，直接返回 URL
     */
    public FastUploadResult fastUpload(MultipartFile file) throws IOException {
        // 1. 计算文件 MD5
        String md5 = DigestUtils.md5DigestAsHex(file.getInputStream());

        // 2. 查询数据库是否已有相同文件
        Optional<FileRecord> existing = fileRecordRepo.findByMd5(md5);
        if (existing.isPresent()) {
            log.info("Fast upload hit: md5={}", md5);
            return FastUploadResult.builder()
                .url(existing.get().getUrl())
                .isFastUpload(true)
                .build();
        }

        // 3. 文件不存在，正常上传
        String url = minioTemplate.uploadFile(file, "files");

        // 4. 保存文件记录
        FileRecord record = FileRecord.builder()
            .md5(md5)
            .url(url)
            .filename(file.getOriginalFilename())
            .size(file.getSize())
            .build();
        fileRecordRepo.save(record);

        return FastUploadResult.builder()
            .url(url)
            .isFastUpload(false)
            .build();
    }
}
```

---

# Part 9: 性能调优与运维

## 9.1 性能调优概览

```
+----------------------------------------------------------+
|               MinIO 性能调优层次                          |
+----------------------------------------------------------+
|  应用层    | 连接池 | 并发上传 | 批量操作 | 流式传输       |
|  JVM 层    | 堆内存 | GC 策略 | NIO     | 线程池         |
|  MinIO 层  | 工作线程 | 连接数 | 压缩   | 存储类         |
|  OS 层     | 文件描述符 | 页缓存 | I/O 调度器        |
|  硬件层    | NVMe SSD | 万兆网络 | RAID 控制器       |
+----------------------------------------------------------+
```

## 9.2 MinIO 服务端调优参数

```bash
# 关键环境变量调优

# 压缩（对文本类文件有显著效果）
MINIO_COMPRESSION_ENABLED=on
MINIO_COMPRESSION_MIME_TYPES=text/*,application/json,application/xml

# 关闭浏览器（生产环境节省资源）
MINIO_BROWSER=off

# 调整扫描频率（默认1分钟，可增大降低 I/O 压力）
MINIO_SCANNER_SPEED=default  # slow/default/fast/max

# 网络
MINIO_API_REQUESTS_MAX=10000  # 最大并发请求数
MINIO_API_REQUESTS_DEADLINE=10s  # 请求排队超时

# 系统级别调优
ulimit -n 65536  # 最大文件描述符
sysctl vm.swappiness=10  # 减少 Swap 使用
sysctl net.core.somaxconn=65535  # TCP 队列
```

## 9.3 Java 客户端调优

```java
// 配置 OkHttpClient 连接池
@Bean
public MinioClient minioClient(MinioProperties props) {
    // 自定义 OkHttpClient
    OkHttpClient httpClient = new OkHttpClient.Builder()
        .connectionPool(new ConnectionPool(
            100,    // 最大空闲连接数
            5,      // 保活时间
            TimeUnit.MINUTES))
        .connectTimeout(10, TimeUnit.SECONDS)
        .writeTimeout(120, TimeUnit.SECONDS)
        .readTimeout(120, TimeUnit.SECONDS)
        .build();

    return MinioClient.builder()
        .endpoint(props.getEndpoint())
        .credentials(props.getAccessKey(), props.getSecretKey())
        .httpClient(httpClient)
        .build();
}

// 并发上传（线程池）
@Service
public class ConcurrentUploadService {

    private final MinioTemplate minioTemplate;
    private final ExecutorService executor =
        Executors.newFixedThreadPool(10); // 10 线程并发上传

    public List<String> uploadBatch(List<MultipartFile> files) {
        List<CompletableFuture<String>> futures = files.stream()
            .map(file -> CompletableFuture.supplyAsync(
                () -> minioTemplate.uploadFile(file, "batch"),
                executor))
            .collect(Collectors.toList());

        return futures.stream()
            .map(CompletableFuture::join)
            .collect(Collectors.toList());
    }
}
```

## 9.4 磁盘与网络 I/O 调优

```bash
# 文件系统选择（推荐 XFS 或 ext4）
mkfs.xfs -f /dev/nvme0n1
mount -o noatime,nodiratime,inode64 /dev/nvme0n1 /data

# I/O 调度器（SSD 使用 none/mq-deadline）
echo none > /sys/block/nvme0n1/queue/scheduler

# 网络调优（高吞吐量场景）
sysctl net.core.rmem_max=268435456    # 接收缓冲区 256MB
sysctl net.core.wmem_max=268435456    # 发送缓冲区 256MB
sysctl net.ipv4.tcp_wmem="4096 65536 268435456"
sysctl net.ipv4.tcp_rmem="4096 65536 268435456"
sysctl net.ipv4.tcp_congestion_control=bbr  # 使用 BBR 拥塞控制
```

## 9.5 Prometheus 监控配置

```bash
# 生成 Prometheus 抓取配置
mc admin prometheus generate myminio > prometheus.yml

# 或者直接在 prometheus.yml 中配置
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: "minio"
    bearer_token: "YOUR_BEARER_TOKEN"
    metrics_path: /minio/v2/metrics/cluster
    scheme: http
    static_configs:
      - targets: ["minio1:9000", "minio2:9000", "minio3:9000", "minio4:9000"]
    scrape_interval: 15s
```

## 9.6 关键监控指标

| 指标名称 | 含义 | 告警阈值参考 |
|---------|------|-------------|
| `minio_bucket_objects_size_bytes` | Bucket 总大小 | 容量 > 80% |
| `minio_s3_requests_total` | S3 请求总数 | 错误率 > 1% |
| `minio_s3_requests_errors_total` | 请求错误数 | 5 分钟内 > 100 |
| `minio_node_disk_used_bytes` | 磁盘使用量 | 磁盘 > 80% |
| `minio_node_disk_free_bytes` | 磁盘剩余量 | < 10GB |
| `minio_inter_node_traffic_recv_bytes` | 节点间接收流量 | 持续 > 1GB/s |
| `minio_node_file_descriptor_open_total` | 打开文件描述符 | > 50000 |

## 9.7 Grafana Dashboard

```
MinIO 官方提供了 Grafana Dashboard（ID: 13502）

导入步骤:
  1. Grafana -> Dashboards -> Import
  2. 输入 Dashboard ID: 13502
  3. 选择 Prometheus 数据源
  4. 点击 Import

关键面板:
  - MinIO Overview: 整体健康状态
  - API 请求延迟: P50/P95/P99 分位数
  - 磁盘 I/O: 读写吞吐量和 IOPS
  - 网络流量: 各节点入/出流量
  - Erasure Healing: 自愈任务进度
```

## 9.8 Kubernetes 部署（Operator）

```yaml
# MinIO Operator Tenant CRD
apiVersion: minio.min.io/v2
kind: Tenant
metadata:
  name: minio-tenant
  namespace: minio-ns
spec:
  image: quay.io/minio/minio:latest
  imagePullPolicy: IfNotPresent

  # 3 个纠删码集（Pool），每个 Pool 4 个节点，每节点 4 个卷
  pools:
    - name: pool-0
      servers: 4
      volumesPerServer: 4
      volumeClaimTemplate:
        metadata:
          name: data
        spec:
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 1Ti   # 每个卷 1TB
          storageClassName: local-nvme

  # 凭证（从 Secret 读取）
  credsSecret:
    name: minio-creds-secret

  # 请求 CPU/内存
  resources:
    requests:
      memory: 8Gi
      cpu: "4"

  # Prometheus 监控
  prometheusOperator: true
```

```bash
# 安装 MinIO Operator
kubectl apply -f https://github.com/minio/operator/releases/latest/download/operator.yaml

# 查看 Operator 状态
kubectl get pods -n minio-operator

# 应用 Tenant
kubectl apply -f tenant.yaml

# 查看 Tenant 状态
kubectl get tenant -n minio-ns
```

## 9.9 常见运维问题排查

### 9.9.1 磁盘满了怎么办

```bash
# 查看磁盘使用情况
mc admin info myminio  # 查看各节点磁盘状态
mc du myminio/mybucket  # 查看 Bucket 占用

# 方法1: 清理过期版本
mc rm --versions --recursive myminio/mybucket/logs/

# 方法2: 设置生命周期规则自动清理
mc lifecycle add myminio/mybucket \
    --prefix logs/ \
    --expiry-days 30

# 方法3: 扩容（添加新的 Pool 而非在线扩容现有 Pool）
mc admin expand myminio http://new-node{1...4}/data{1...4}
```

### 9.9.2 节点宕机恢复

```bash
# 查看集群健康状态
mc admin status myminio

# 查看自愈任务进度
mc admin heal -r myminio/  # -r 表示递归

# 替换故障节点后，触发全量自愈
mc admin heal --recursive --force-start myminio/
```

### 9.9.3 上传失败常见原因

| 错误信息 | 原因 | 解决方案 |
|---------|------|----------|
| `BucketAlreadyOwnedByYou` | Bucket 已存在 | 直接使用该 Bucket |
| `EntityTooLarge` | 对象超过 5TB 限制 | 联系 MinIO 企业版 |
| `InvalidBucketName` | Bucket 名称不合法 | 只用小写字母/数字/- |
| `NoSuchBucket` | Bucket 不存在 | 先创建 Bucket |
| `AccessDenied` | 权限不足 | 检查 IAM 策略 |
| `RequestTimeout` | 请求超时 | 增大客户端超时配置 |
| `SlowDown` | 请求限流 | 降低并发，设置 backoff |

### 9.9.4 数据一致性检查

```bash
# 检查对象完整性
mc support inspect myminio/mybucket/object-key

# 全量一致性检查（耗时长，在空闲时执行）
mc admin heal --recursive --scan deep myminio/

# 检查磁盘健康
mc admin drive scan myminio
```

## 9.10 数据迁移

```bash
# MinIO 到 MinIO 迁移（使用 mc mirror）
mc alias set source http://old-minio:9000 admin password
mc alias set dest   http://new-minio:9000 admin password

mc mirror source/mybucket dest/mybucket --overwrite --remove

# AWS S3 到 MinIO 迁移
mc alias set s3 https://s3.amazonaws.com ACCESS_KEY SECRET_KEY
mc mirror s3/my-s3-bucket myminio/my-local-bucket

# 迁移并实时同步（--watch 参数）
mc mirror --watch source/mybucket dest/mybucket
```

---

# Part 10: 面试题 FAQ

## Q1：MinIO 和 HDFS 有什么区别？什么场景下选择 MinIO？

```
+--------------------+----------------+------------------+
| 维度               | MinIO          | HDFS             |
+--------------------+----------------+------------------+
| 设计目标           | 对象存储（S3） | 大文件批量处理   |
| 最小文件粒度       | 无限制         | 推荐 > 64MB      |
| API 类型           | RESTful S3     | HDFS Native API  |
| 小文件支持         | 良好           | 差               |
| 生态兼容           | S3 生态        | Hadoop 生态      |
| 部署复杂度         | 极低           | 高               |
| 元数据服务         | 分布式内置     | NameNode（单点） |
+--------------------+----------------+------------------+

选择 MinIO 的场景:
  - 需要 S3 兼容 API 的云原生应用
  - 小文件/媒体文件存储（头像、文档）
  - AI/ML 训练数据集
  - 现有 AWS S3 迁移到私有化
```

## Q2：MinIO 纠删码和 RAID 有什么区别？

```
RAID-5/RAID-6 的局限性:
  - RAID-6 最多容忍 2 块磁盘故障
  - RAID 在重建过程中会产生"写洞"问题
  - RAID 绑定硬件控制器，无法跨节点扩展
  - RAID 重建期间性能严重下降

MinIO 纠删码的优势:
  - 可配置任意数量的校验块（EC:2 到 EC:N/2）
  - 基于软件实现，无需专用硬件
  - 分布式存储，可跨节点、跨机架容错
  - 重建期间仍可正常提供服务
  - 结合 HighwayHash 位衰减检测，RAID 无此能力
```

## Q3：MinIO 如何实现高可用？如果一个节点宕机会怎样？

```
高可用机制:

  1. 读仲裁（Read Quorum）: 读取时需要 data_shards 个节点响应
  2. 写仲裁（Write Quorum）: 写入时需要 data_shards + 1 个节点成功
  3. 纠删码恢复: 即使部分节点宕机，现有数据仍可从其他分片恢复

以 8 节点 EC:4 为例（4 数据块 + 4 校验块）:
  - 读取仲裁: 需要 4 个节点
  - 写入仲裁: 需要 5 个节点
  - 最多允许 4 个节点同时故障仍可读取
  - 最多允许 3 个节点同时故障仍可写入

节点宕机后:
  - 少于仲裁数量的节点宕机: 服务正常，后台自愈数据
  - 超过仲裁数量的节点宕机: 读/写服务降级或中断
```

## Q4：MinIO 预签名 URL 的实现原理是什么？

```
预签名 URL 原理:

  URL 格式:
  http://minio:9000/bucket/object
    ?X-Amz-Algorithm=AWS4-HMAC-SHA256
    &X-Amz-Credential=ACCESS_KEY/20240115/us-east-1/s3/aws4_request
    &X-Amz-Date=20240115T103000Z
    &X-Amz-Expires=3600
    &X-Amz-SignedHeaders=host
    &X-Amz-Signature=<HMAC-SHA256签名>

  签名算法 (AWS Signature V4):
  1. 构造 StringToSign = 请求方法 + 日期 + 路径 + 查询参数的哈希
  2. 派生签名密钥 = HMAC(HMAC(HMAC(HMAC("AWS4"+SecretKey, date), region), service), "aws4_request")
  3. Signature = HMAC(SigningKey, StringToSign)

  安全性:
  - URL 中的 Signature 是用 SecretKey 签名的，无法伪造
  - Expires 参数控制有效期，过期后 MinIO 拒绝请求
  - 可以绑定特定 IP（通过添加 aws:SourceIp 条件）
```

## Q5：MinIO 版本控制开启后会额外占用多少存储空间？

```
版本控制的存储影响:

  每次写入（PUT）都会创建新版本，每个版本占用完整的对象空间。
  例如: 同一对象写入 10 次 (每次 1MB) = 10MB 存储

控制存储开销的策略:
  1. 生命周期规则: 设置非最新版本的过期时间
     mc lifecycle add myminio/bucket \
         --noncurrent-expiry-days 30  # 非最新版本30天后删除

  2. 保留最近 N 个版本:
     mc lifecycle add myminio/bucket \
         --noncurrent-versions 5  # 仅保留最近5个版本

  3. 定期清理删除标记:
     mc lifecycle add myminio/bucket --expired-object-delete-marker on
```

## Q6：MinIO 集群扩容时需要注意什么？

```
MinIO 扩容的限制和注意事项:

  1. 不能修改现有 Pool 的节点数量（因为这会破坏现有纠删码组）
  2. 扩容只能通过添加新的 Pool（Server Pool）来实现
  3. 新 Pool 的节点数和磁盘数可以与现有 Pool 不同
  4. 扩容后新数据会按照容量比例分配到各 Pool
  5. 历史数据不会自动迁移到新 Pool（需要手动 mirror）

推荐做法:
  - 规划好初始容量，预留扩容余量
  - 使用相同规格的节点和磁盘保证性能一致性
  - 扩容前先在测试环境验证
```

## Q7：MinIO 如何防止数据被篡改？

```
MinIO 的数据完整性保护体系:

  1. 传输层安全 (TLS)
     客户端 <--TLS--> MinIO: 防止传输中篡改

  2. ETag 校验
     每个对象上传后返回 ETag（MD5 或分片 MD5），
     客户端可对比 ETag 验证数据完整性

  3. HighwayHash 位衰减检测
     每个存储分片都有 HighwayHash-256 校验和，
     读取时自动验证，发现损坏立即纠删恢复

  4. 对象锁（WORM）
     开启 Object Lock 后，在保留期内任何人都无法修改

  5. 版本控制
     每次写入都产生新版本，删除只创建删除标记，历史数据可恢复
```

## Q8：MinIO 的 Bucket Policy 和 IAM Policy 有什么区别？

```
+------------------+----------------------+----------------------+
| 维度             | Bucket Policy        | IAM Policy           |
+------------------+----------------------+----------------------+
| 作用对象         | 绑定到 Bucket        | 绑定到用户/组/角色   |
| Principal        | 可指定所有人(*)      | 隐含当前用户         |
| 跨账号访问       | 支持                 | 需要配合 Bucket Policy|
| 匿名访问         | 支持                 | 不适用               |
| 典型用途         | 公共读、IP 限制      | 用户权限管理         |
+------------------+----------------------+----------------------+

权限判断顺序（白名单优先）:
  1. 明确 Deny -> 拒绝
  2. Bucket Policy 明确 Allow -> 允许
  3. IAM Policy 明确 Allow -> 允许
  4. 默认 -> 拒绝
```

## Q9：如何处理 MinIO 上传超大文件（> 5GB）时的性能问题？

```
超大文件上传最佳实践:

  1. 分片上传（Multipart Upload）
     - 推荐分片大小: 64MB ~ 256MB（根据网络带宽调整）
     - MinIO 最大支持 10000 个分片，每片最大 5GB
     - 对象最大支持 5TB

  2. 并发分片上传
     - 同时上传多个分片（建议并发数 4~8）
     - 需要合理控制并发避免带宽饱和

  3. 断点续传
     - 将 uploadId 和已完成 Part 信息存储到 Redis
     - 网络中断后重新连接，跳过已上传的分片

  4. 前端直传（绕过后端）
     - 后端只生成预签名 URL，前端直接上传到 MinIO
     - 避免后端成为带宽瓶颈
```

## Q10：MinIO 生产环境推荐的最小集群规模是多少？

```
MinIO 生产最小规模推荐:

  最小配置（EC:2, 4节点）:
    - 4 台服务器，每台 4 块 NVMe SSD
    - 纠删码: 4 数据块 + 2 校验块 (EC:2)
    - 可容忍 2 节点故障
    - 最大可用容量: 总容量 * (4-2)/4 = 50%

  推荐配置（EC:4, 8节点）:
    - 8 台服务器，每台 4 块 NVMe SSD
    - 纠删码: 4 数据块 + 4 校验块 (EC:4)
    - 可容忍 4 节点故障
    - 最大可用容量: 总容量 * 50%

  硬件选型:
    - CPU: 16 核以上（EC 计算密集）
    - 内存: 64GB 以上（对象元数据缓存）
    - 磁盘: NVMe SSD（高 IOPS）或 大容量 HDD（高吞吐）
    - 网络: 万兆以太网（10GbE 或更高）
    - 专用网络: 节点间通信使用独立网卡
```

## Q11：MinIO 的 SLA 和 AWS S3 相比如何？

```
+------------------+----------------------+---------------------+
| 指标             | MinIO MNMD (8节点)   | AWS S3              |
+------------------+----------------------+---------------------+
| 可用性 SLA       | 自己运维决定         | 99.99%              |
| 持久性           | 11个9（配置EC:4）    | 11个9               |
| RPO              | 0 (同步复制)         | 0 (同步)            |
| RTO              | 分钟级（宕机恢复）   | 立即                |
| 跨区域复制       | 需要手动配置 mirror  | 内置                |
| 成本             | 硬件 + 运维          | 按使用量付费         |
+------------------+----------------------+---------------------+

MinIO 适合: 私有化部署、数据合规要求、成本敏感场景
AWS S3 适合: 无运维压力、弹性扩容、全球多区域场景
```

## Q12：为什么 MinIO 官方不推荐 NFS 作为存储后端？

```
MinIO 不推荐 NFS 的原因:

  1. POSIX 语义问题
     MinIO 依赖原子性 rename 操作来保证写入一致性，
     NFS 的 rename 操作不是原子的，在网络抖动时可能导致数据不一致

  2. 文件锁问题
     MinIO 使用文件锁（flock/fcntl）来防止并发写入冲突，
     NFS 的文件锁实现不可靠

  3. 性能瓶颈
     NFS 引入了网络层，增加了延迟，无法充分发挥本地 NVMe SSD 的性能

  4. 元数据操作慢
     MinIO 大量使用目录遍历和文件元数据操作，NFS 上这些操作性能差

推荐: 使用本地磁盘（ext4/XFS 格式），通过 MinIO 自身的分布式架构实现冗余
```

## Q13：MinIO 的对象命名有什么限制？

```
对象名称规则:

  - 最大长度: 1024 字节 (UTF-8)
  - 支持: 字母、数字、/、-、_、.、~
  - 路径分隔符: / (MinIO 不真正创建目录，/ 只是前缀)
  - 不推荐: 以 / 开头、连续 //、特殊控制字符

Bucket 名称规则:

  - 长度: 3 ~ 63 字符
  - 只允许: 小写字母、数字、连字符 -
  - 必须以字母或数字开头/结尾
  - 不能是 IP 地址格式
  - 不能包含连续两个连字符 --

Java 中的编码处理:
  String objectName = URLEncoder.encode(originalName, "UTF-8")
                       .replace("+", "%20");
```

## Q14：如何实现 MinIO 与数据库的事务一致性？

```java
// 问题：文件上传到 MinIO 成功，但数据库保存失败 --> 数据不一致

// 解决方案：先写数据库（PENDING状态），再上传 MinIO，最后更新状态

@Service
@Transactional
public class FileConsistencyService {

    @Transactional
    public String uploadFileConsistently(MultipartFile file, Long userId) {
        // 1. 先在数据库创建 PENDING 记录
        String objectKey = "files/" + UUID.randomUUID() + getExt(file);
        FileRecord record = FileRecord.builder()
            .objectKey(objectKey)
            .status(FileStatus.PENDING)
            .userId(userId)
            .build();
        fileRecordRepo.save(record);  // 数据库事务

        try {
            // 2. 上传到 MinIO
            minioTemplate.uploadFile(file, objectKey);

            // 3. 更新数据库状态为 DONE
            record.setStatus(FileStatus.DONE);
            record.setUrl(minioTemplate.getPublicUrl(objectKey));
            fileRecordRepo.save(record);

            return record.getUrl();
        } catch (Exception e) {
            // 4. MinIO 上传失败，更新状态为 FAILED
            record.setStatus(FileStatus.FAILED);
            fileRecordRepo.save(record);
            // 启动异步补偿任务（清理 MinIO 上可能上传的部分数据）
            asyncCleanup(objectKey);
            throw new RuntimeException("Upload failed", e);
        }
    }

    // 补偿任务：清理 PENDING 超过 1 小时的孤立记录
    @Scheduled(fixedDelay = 3600000)
    public void cleanupPendingFiles() {
        LocalDateTime threshold = LocalDateTime.now().minusHours(1);
        List<FileRecord> pendingFiles = fileRecordRepo
            .findByStatusAndCreatedAtBefore(FileStatus.PENDING, threshold);

        for (FileRecord record : pendingFiles) {
            try {
                if (minioTemplate.objectExists(record.getObjectKey())) {
                    minioTemplate.deleteFile(record.getObjectKey());
                }
                record.setStatus(FileStatus.EXPIRED);
                fileRecordRepo.save(record);
            } catch (Exception e) {
                log.error("Cleanup failed for: {}", record.getObjectKey(), e);
            }
        }
    }
}
```

---

# 附录：MinIO 快速参考

## A.1 常用 mc 命令速查

```bash
# ===== 别名管理 =====
mc alias set <alias> <url> <access-key> <secret-key>
mc alias list
mc alias remove <alias>

# ===== Bucket 操作 =====
mc mb <alias>/<bucket>                    # 创建
mc ls <alias>/                            # 列出
mc rb <alias>/<bucket>                   # 删除（必须为空）
mc rb --force <alias>/<bucket>           # 强制删除（含内容）
mc anonymous set <perm> <alias>/<bucket> # 设置公开权限 [none|download|upload|public]

# ===== 对象操作 =====
mc ls <alias>/<bucket>/                  # 列出对象
mc ls --recursive <alias>/<bucket>/      # 递归列出
mc cp <src> <dst>                        # 复制
mc mv <src> <dst>                        # 移动
mc rm <alias>/<bucket>/<object>          # 删除对象
mc rm --recursive <alias>/<bucket>/logs/ # 递归删除
mc mirror <src> <dst>                    # 同步目录
mc mirror --watch <src> <dst>            # 实时同步
mc stat <alias>/<bucket>/<object>        # 对象信息
mc cat <alias>/<bucket>/<object>         # 读取内容
mc head -n 10 <alias>/<bucket>/<object>  # 读取前10行

# ===== 版本管理 =====
mc version enable <alias>/<bucket>       # 开启版本控制
mc ls --versions <alias>/<bucket>/       # 列出所有版本
mc rm --version-id <id> <alias>/<bucket>/<object>  # 删除指定版本

# ===== 生命周期 =====
mc lifecycle add <alias>/<bucket> --expiry-days 30
mc lifecycle ls <alias>/<bucket>
mc lifecycle remove <alias>/<bucket>

# ===== 管理命令 =====
mc admin info <alias>                    # 集群信息
mc admin status <alias>                  # 节点状态
mc admin heal -r <alias>/                # 自愈修复
mc admin top disks <alias>               # 磁盘 I/O 监控
mc admin top api <alias>                 # API 调用监控
mc admin decommission start <alias> <pool-url>  # 下线 Pool
mc admin update <alias>                  # 升级 MinIO
mc admin trace <alias>                   # 实时请求追踪
mc admin logs <alias>                    # 实时日志
```

## A.2 Java SDK 常用对象速查

| 操作 | 方法 | 参数类 |
|------|------|--------|
| 创建 Bucket | `makeBucket()` | `MakeBucketArgs` |
| 检查 Bucket | `bucketExists()` | `BucketExistsArgs` |
| 上传对象 | `putObject()` | `PutObjectArgs` |
| 下载对象 | `getObject()` | `GetObjectArgs` |
| 对象信息 | `statObject()` | `StatObjectArgs` |
| 删除对象 | `removeObject()` | `RemoveObjectArgs` |
| 列出对象 | `listObjects()` | `ListObjectsArgs` |
| 复制对象 | `copyObject()` | `CopyObjectArgs` |
| 预签名 URL | `getPresignedObjectUrl()` | `GetPresignedObjectUrlArgs` |
| POST 表单 | `getPresignedPostFormData()` | `PostPolicy` |
| 初始化分片 | `createMultipartUpload()` | -- |
| 上传分片 | `uploadPart()` | -- |
| 完成分片 | `completeMultipartUpload()` | -- |
| 取消分片 | `abortMultipartUpload()` | -- |

## A.3 MinIO 存储目录结构

```
/data/                          <- 数据根目录
  .minio.sys/                   <- MinIO 系统数据（勿删）
    buckets/
      bucket-name/
        .metadata.bin           <- Bucket 元数据
  bucket-name/                  <- 用户 Bucket
    object-prefix/
      object-name/
        xl.meta                 <- 对象元数据（MessagePack）
        part.1                  <- 数据分片
        part.2                  <- 多 part 时的第2片
```

---

> **注意**: 本文档基于 MinIO RELEASE.2024 版本编写，部分 API 和命令行语法
> 可能随版本更新而变化，请以官方文档为准: https://min.io/docs/minio/linux/

> 更多内容请参考:
> - MinIO GitHub: https://github.com/minio/minio
> - Java SDK: https://github.com/minio/minio-java
> - MinIO mc: https://github.com/minio/mc

