---
title: Apache Fesod 实战 — 新一代 Java Excel 处理神器
slug: apache-fesod-excel-processing
description: >-
  Apache Fesod——从 EasyExcel 继承衣钵，由原作者打造的 Apache 孵化器项目。
  本文带你了解它的前世今生、核心架构、性能表现，以及从 EasyExcel 迁移的实战指南。
tags:
  - technical
added: "July 22 2026"
---

## 引言

做 Java 开发，谁没跟 Excel 打过交道？

早些年 Apache POI 是唯一的选择，API 繁琐不说，百万行数据就能把堆撑到 OOM。后来阿里巴巴开源的 EasyExcel 凭借「一行代码读 Excel」的简洁体验和流式处理能力，迅速成为 Java 生态的事实标准。

但 2024 年底，阿里宣布 EasyExcel 进入维护模式——有新 Bug 才修，没新功能。

故事还没完。EasyExcel 的原作者离开阿里后，很快推出了 **FastExcel**，并在 2025 年捐献给 Apache 孵化器，正式更名为 **Apache Fesod**（Fast. Easy. Done.）。2026 年 2 月发布 2.0.1-incubating，近期又迭代到 2.0.2，标志着 Java Excel 处理迈入全新时代。

本文从演进历史、核心架构、性能到实战迁移，一次性讲透。

## 一、从 POI 到 Fesod：三十年 Excel 史

### 1.1 Apache POI 时代（2001–2018）

POI 垄断 Java Excel 处理近 20 年，底层基于 DOM 模型——读取时将整个文件加载到内存，构建完整的对象树。问题是：

- 10 万行 Excel 占用 2GB+ 堆内存
- 大文件频繁 OOM，线上导出服务三天两头重启
- API 极重，`HSSFWorkbook`、`XSSFWorkbook`、`Row`、`Cell`…… 读个简单表格要写 20 行代码
- Bug 多，尤其内存泄漏

POI 不是不好，只是用在了错误的地方——它适合创建复杂格式的报表，不适合做大数据量的批处理。

### 1.2 EasyExcel 时代（2018–2024）

阿里 EasyExcel 做了两件关键的事：

1. **流式读写** —— 基于 POI SAX 模式，按行解析，不把所有数据加载到内存
2. **注解驱动** —— `@ExcelProperty` 一行注解完成字段映射，API 极简

结果：处理 100 万行数据只需几十 MB 内存，API 调用从几十行缩到三五行。EasyExcel 迅速流行，成为 Spring Boot 生态标配。

但 EasyExcel 有一个痛点：底层强依赖 POI，而 POI 各个版本之间有不兼容的改动，导致 EasyExcel 在升级 POI 时处处掣肘。另外作为 Ali 内部项目，外部贡献者的接入流程也不顺畅。

### 1.3 Fesod 时代（2025–至今）

2024 年底，原作者从阿里离职，将 EasyExcel 的重构版本 FastExcel 独立出来。2025 年 5 月正式进入 Apache 孵化器，更名为 **Apache Fesod**。

这是一个罕见的「三连跳」：

```
EasyExcel（阿里内部） → FastExcel（原作者独立） → Apache Fesod（Apache 孵化器）
```

Fesod 的定位非常明确：**在保持 EasyExcel 简洁 API 的基础上，完全重构底层，追求极致的性能和内存控制。**

项目地址：[https://github.com/apache/fesod](https://github.com/apache/fesod)

## 二、Fesod 的核心架构

Fesod 解决的核心问题只有一个：**如何用最少的内存，最快地读写 Excel 文件。**

### 2.1 流式处理架构

POI 读取 Excel 的传统方式是 DOM（Document Object Model）：

```
Excel 文件 → 完全解析 → 内存中构建完整对象树 → 应用层处理
```

Fesod 采用 SAX（Simple API for XML）+ 事件驱动模型：

```
Excel 文件 → SAX 解析器 → 逐行解析 → 按行回调 ReadListener → 处理完即丢弃
```

这个差异在大文件下是数量级的——POI 需要保留整棵 XML 树，Fesod 只需要保留当前行。

### 2.2 智能缓存机制

Fesod 引入了多层缓存架构：

- **行级缓存** —— 当前批次数据保持在内存，处理完后释放
- **文件级缓存** —— 利用 Ehcache 3.x，将频繁访问的元数据（样式、共享字符串表）缓存到磁盘或堆外内存
- **共享字符串去重** —— Excel 中的共享字符串表（Shared String Table）是内存大户，Fesod 对 SST 做了压缩和延迟加载，只有被引用的字符串才会加载

### 2.3 零拷贝写入

写入时，Fesod 采用「流式写入 + 文件分片」策略：

- 数据不经过 List 缓存，直接流式写入临时文件
- 最终通过文件合并完成 XLXS 的 ZIP 结构组装
- 写入 100 万行数据，峰值内存不超过 64MB

## 三、性能对比

以下是社区公开的压测数据（测试环境：i7-12700, 32GB RAM, JDK 17）：

### 读取性能

| 数据量    | POI 5.x      | EasyExcel 3.x | Fesod 2.0.x |
|-----------|-------------|---------------|-------------|
| 10 万行   | 8.2s/2.1GB  | 3.1s/256MB    | **2.8s/42MB**  |
| 100 万行  | OOM         | 28s/1.1GB     | **25s/47MB**   |
| 500 万行  | OOM         | OOM/6GB+      | **130s/47MB**  |

### 写入性能

| 数据量    | POI 5.x      | EasyExcel 3.x | Fesod 2.0.x |
|-----------|-------------|---------------|-------------|
| 10 万行   | 6.5s/1.8GB  | 2.1s/180MB    | **1.8s/35MB**  |
| 100 万行  | OOM         | 18s/780MB     | **16s/42MB**   |
| 500 万行  | OOM         | OOM           | **87s/50MB**   |

**关键结论：** Fesod 的内存占用比 EasyExcel 低 5–7 倍，比 POI 低 40–80 倍。500 万行数据稳定在 50MB 以内，写场景也一样。

## 四、快速上手

### 4.1 引入依赖

最新版已发布到 Maven Central：

```xml
<dependency>
    <groupId>org.apache.fesod</groupId>
    <artifactId>fesod-sheet</artifactId>
    <version>2.0.2-incubating</version>
</dependency>
```

> **注意：** 如果使用 `fesod-bom` 统一管理版本，请留意 GAV 坐标的变化——部分早期孵化版本的 groupId/artifactId 有过调整。如果遇到找不到包的情况，确认仓库已刷新至最新中央仓库索引。

**如果你是 Gradle 用户：**

```gradle
implementation 'org.apache.fesod:fesod-sheet:2.0.2-incubating'
```

### 4.2 定义 POJO

```java
public class DemoData {
    @ExcelProperty("姓名")
    private String name;

    @ExcelProperty("年龄")
    private Integer age;

    @ExcelProperty("邮箱")
    private String email;

    // getter / setter 省略
}
```

`@ExcelProperty` 的 value 与 Excel 表头名称匹配，也可以用 index 按列号匹配：

```java
@ExcelProperty(index = 0)
private String name;
```

### 4.3 读取 Excel

**同步读取（小文件）：**

```java
List<DemoData> list = FesodSheet.read("demo.xlsx")
    .head(DemoData.class)
    .sheet()
    .doReadSync();
```

**流式读取（大文件 + 监听器）：**

```java
FesodSheet.read("large-file.xlsx")
    .head(DemoData.class)
    .registerReadListener(new ReadListener<DemoData>() {
        @Override
        public void invoke(DemoData data, AnalysisContext context) {
            System.out.println("读取一行: " + data);
        }

        @Override
        public void doAfterAllAnalysed(AnalysisContext context) {
            System.out.println("全部读取完成");
        }
    })
    .sheet()
    .doRead();
```

流式读取的关键在于 `ReadListener`——每读一行回调一次，处理完立即释放，堆里永远不会堆积。

### 4.4 写入 Excel

```java
List<DemoData> dataList = Arrays.asList(
    new DemoData("张三", 25, "zhangsan@example.com"),
    new DemoData("李四", 30, "lisi@example.com")
);

FesodSheet.write("output.xlsx", DemoData.class)
    .sheet("用户信息")
    .doWrite(dataList);
```

### 4.5 读取指定 Sheet

```java
// 读取第一个 sheet
FesodSheet.read("demo.xlsx")
    .head(DemoData.class)
    .sheet()
    .doReadSync();

// 读取名为 "Sheet2" 的 sheet
FesodSheet.read("demo.xlsx")
    .head(DemoData.class)
    .sheet("Sheet2")
    .doReadSync();

// 读取第 0 个 sheet
FesodSheet.read("demo.xlsx")
    .head(DemoData.class)
    .sheet(0)
    .doReadSync();
```

## 五、从 EasyExcel / FastExcel 迁移

对于已经在使用 EasyExcel 或 FastExcel 的项目，迁移到 Fesod 基本是无痛的。API 保持高度兼容，主要改动集中在：

### 5.1 依赖替换

```xml
<!-- 原来：EasyExcel -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>easyexcel</artifactId>
    <version>3.3.4</version>
</dependency>

<!-- 原来：FastExcel -->
<dependency>
    <groupId>cn.idev.excel</groupId>
    <artifactId>fastexcel</artifactId>
    <version>1.3.0</version>
</dependency>

<!-- 现在：Fesod -->
<dependency>
    <groupId>org.apache.fesod</groupId>
    <artifactId>fesod-sheet</artifactId>
    <version>2.0.2-incubating</version>
</dependency>
```

### 5.2 导包替换

**全局替换 import 路径：**

| 原库        | 旧包名               | 新包名                      |
|------------|---------------------|----------------------------|
| EasyExcel  | `com.alibaba.excel.*` | `org.apache.fesod.sheet.*` |
| FastExcel  | `cn.idev.excel.*`     | `org.apache.fesod.sheet.*` |

### 5.3 入口类替换

| 原库        | 旧入口                    | 新入口                          |
|------------|-------------------------|-------------------------------|
| EasyExcel  | `EasyExcel.read/write`  | `FesodSheet.read/write`       |
| FastExcel  | `FastExcel.read/write`  | `FesodSheet.read/write`       |

### 5.4 注解兼容

`@ExcelProperty`、`@ColumnWidth`、`@ContentFontStyle`、`@ContentRowHeight`、`@HeadFontStyle`、`@HeadRowHeight` 等注解全部兼容，只是包名从 `com.alibaba.excel.annotation` 或 `cn.idev.excel.annotation` 迁移到 `org.apache.fesod.sheet.annotation`。

### 5.5 迁移 Checklist

```
□ 替换 pom.xml 或 build.gradle 中的依赖坐标
□ 全局替换 import 语句（IDE 一键替换）
□ 替换入口类调用（EasyExcel → FesodSheet）
□ 运行单元测试，验证读、写、多 Sheet 功能
□ 验证合并单元格、样式、图片等高级功能
□ 回归 Web 接口的上传下载功能
```

对于大多数项目，迁移时间在 **30 分钟到 2 小时** 之间，取决于 Excel 功能的使用深度。

## 六、高级用法

### 6.1 日期与数字格式化

```java
public class OrderData {
    @ExcelProperty("订单号")
    private String orderId;

    @ExcelProperty(value = "下单时间")
    private Date orderTime;

    @ExcelProperty(value = "金额")
    @NumberFormat("#,##0.00")
    private BigDecimal amount;
}
```

### 6.2 排除字段

```java
@ExcelProperty(value = "内部备注", exclude = true)
private String internalNote;
```

### 6.3 自定义样式

```java
FesodSheet.write("styled.xlsx", DemoData.class)
    .registerWriteHandler(new CustomCellStyleHandler())
    .sheet("样式示例")
    .doWrite(dataList);
```

### 6.4 大文件分批写入（防止 OOM）

```java
ExcelWriter excelWriter = FesodSheet.write("batch.xlsx", DemoData.class)
    .build();
WriteSheet writeSheet = FesodSheet.writerSheet("数据").build();

// 分批写入，每批 10 万行
for (int i = 0; i < totalBatches; i++) {
    List<DemoData> batch = fetchBatch(i);
    excelWriter.write(batch, writeSheet);
    batch.clear(); // 让 GC 及时回收
}

excelWriter.finish();
```

### 6.5 使用 BOM 统一版本管理

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.apache.fesod</groupId>
            <artifactId>fesod-bom</artifactId>
            <version>2.0.2-incubating</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 七、与 POI 对比：什么时候选谁？

| 维度          | Apache POI                    | Apache Fesod                      |
|--------------|-------------------------------|-----------------------------------|
| API 复杂度   | 高，需要手动管理 Workbook/Row/Cell | 低，注解驱动 + 链式调用              |
| 大文件读取   | ❌ 100 万行 OOM                | ✅ 500 万行稳定 50MB                |
| 写入性能     | 中等，内存随数据量线性增长        | 优秀，流式写入几乎恒定内存            |
| 格式支持     | 极全（Word、PPT、Visio 等）     | 专注 Excel/CSV，小而精              |
| 样式支持     | 非常丰富                        | 基本覆盖，复杂报表需配合 POI          |
| 社区活跃度   | 成熟但创新缓慢                   | Apache 孵化器，迭代迅速              |

**建议：**

- **简单读写 + 大数据量** → 首选 Fesod
- **复杂 Word/PPT 操作** → POI 仍是唯一选择
- **复杂 Excel 报表 + 自定义 XML** → 可以 Fesod + POI 混合使用

## 八、现状与展望

Fesod 目前处在 Apache 孵化阶段（Incubating），意味着：

- ✅ 社区代码审查和贡献流程已建立
- ✅ 发布到 Maven Central，Apache 签名认证
- ✅ 官方文档中英双语，示例齐全
- ⏳ 进入 Apache TLP（顶级项目）只是时间问题

值得关注的是，Fesod 的底层核心已经完全重写，不再像 EasyExcel 那样强绑 POI 版本。这意味着未来 Fesod 可以独立优化底层 XML 解析、流式处理、内存管理等核心路径，而不受 POI 更新的牵制。

从路线图来看，未来计划包括：

- CSV 原生支持（不依赖 Commons CSV）
- 跨平台 SDK（非 JVM 平台）
- 更细粒度的内存监控 API
- 云原生场景下的分布式 Excel 处理支持

## 九、总结

Apache Fesod 的出现，可以看作是 Java Excel 处理领域的一次「正本清源」——原作者带着从 EasyExcel 积累的宝贵经验，在一个更开放、更中立的基金会下，从头重构了一款真正的高性能 Excel 库。

对于还在用 POI 或者老旧 EasyExcel 版本的项目，迁移到 Fesod 几乎没有成本，却能换来 **数量级的内存节省和更活跃的社区支持**。如果你的项目即将上线或已经在大规模使用 Excel 处理，Fesod 值得你现在就尝试。

**资源链接：**

- GitHub：https://github.com/apache/fesod
- 官方文档：https://fesod.apache.org
- Maven Central：`org.apache.fesod:fesod-sheet:2.0.2-incubating`
