---
layout: post
title: "Lucene Codec 自定义扩展实现"
categories: blog
tags: ["Lucene", "Lucene Codec", "信息检索"]
---

Codec 是 Lucene 4.0 之后编解码索引信息的一种机制，它解耦了 Lucene 内部复杂的搜索/索引数据结构和外部存储之间的关系。通过对 Codec API 的扩展，用户可以方便的自定义 Lucene 索引数据的编码方式和存储结构。

**Lucene Codec 分类**

Lucene Codec 按段（Segment）配置，每个段都可以配置不同的 Codec。Codec 又将段内容分成了 9 个部分，分别定义了不同的 Format 类，用户可以通过继承不同的 Format 类实现对索引不同的部分进行自定义编解码。Codec 为每个 Format 类都暴露了相关的读/写 API，分别用于搜索数据的读取和索引数据的写入。

Codec 定义的 9 种 Format 如下：
1. **StoredFieldsFormat**：编解码每个文档的存储字段。
2. **TermVectorsFormat**：编解码单个文档的词向量（term vectors）。
3. **LiveDocsFormat**：处理一个可选的 bitset，用于标记哪些文档未被删除。
4. **NormsFormat**：编解码索引时的打分归一化因子。
5. **SegmentInfoFormat**：存储段的元数据信息，如：段名称、包含的文档数、唯一 id 等。
6. **FieldInfosFormat**：记录单个字段的元数据信息，如：字段名称、字段是否被索引、是否被存储等。
7. **PostingsFormat**：涵盖了倒排索引所有相关内容，包括所有字段，词项，文档，位置，payloads 以及偏移量。查询的 Queries 使用这个 API 查找其匹配的文档。
8. **DocValuesFormat**：处理每个文档的列式存储数据。默认 format 实现存储数据在磁盘并在内存中加载其索引以便于快速的随机访问。
9. **CompoundFormat**：在 compound 文件格式打开的情况下使用（默认开启）。它将上述所有格式写入的多个单独文件合并为每个段两个文件，以减少搜索过程中所需的文件句柄数。

注意：由于 postings 和 doc values  两个格式特别重要，且每个字段间这两个格式都可能产生变化。因此这两个格式有特殊的 per-field format 用于区分相同段之间的不同字段，允许段内每个字段都拥有不同的格式。实现时可以通过 per-field 的接口返回不同字段 Format。

**关于 FilterCodec**

很多时候用户并不需要扩展全部 Codec Format，只需扩展自己需要的几个 Format 就行，其他 Format 采用 Lucene 默认实现。

Lucene 内置提供了一个 FilterCodec 抽象类，可以将其所有方法调用转发至另一个 codec 实现类。用户自定义扩展部分 Format 时，只需继承 FilterCodec，指定默认实现为 LuceneXXCodec，重写相关的 Format 接口并返回自定义 Format 类即可。

例如，基于 FilterCodec 重定义 Lucene MN 版本的 liveDocsFormat 实现：
```java
public final class CustomCodec extends FilterCodec {
  
    public CustomCodec() {
        super("CustomCodec", new LuceneMNCodec());
    }
  
    public LiveDocsFormat liveDocsFormat() {
        return new CustomLiveDocsFormat();
    }
  
}
```

**Lucene Codec 注册机制**

Lucene Codec 使用 Java SPI 机制注册自定义实现类，且每个 Codec 都有一个唯一的名字，比如 "Lucene410"。SPI 通过 Codec 的名称查找/加载相应实例。所有的 Codec 都是抽象类 org.apache.lucene.codecs.Codec 的实例。因此，根据 SPI 规范，需要在资源目录 META-INF/services 下放置 service provider 配置文件，配置文件名为抽象类名 "org.apache.lucene.codecs.Codec"，内容为自定义实现类全名。

Lucene Codec 注册具体实现：

step-1：  
创建项目，（以 Maven 为例）并在 pom 文件中添加 lucene-core、lucene-codec 依赖。

step-2：  
将自定义实现的 codec 类发布至 META-INF/services 目录。  
1. 在 src/main/resources/services 目录下创建文件 org.apache.lucene.codecs.Codec  
2. 文件中写入自定义实现类带包名的全名，例如：com.netease.panther.codec.MyCodec  
3. 在 Maven 中配置 build-resource 将配置文件拷贝至目标 target/META-INF 目录：

```java
<build>
   <resources>
       <resource>
           <directory>src/main/resources/services</directory>
           <targetPath>META-INF/services</targetPath>
       </resource>
   </resources>
 </build>
```

**扩展 DocValuesFormat 具体实现**

> 以 Lucene 8.0 版本为例，扩展其 DocValuesFormat 实现。

1、自定义 Codec 实现类
```java
public final class MyCodec extends FilterCodec {

    private final MyDocValuesFormat myDocValuesFormat = new MyDocValuesFormat();

    public MyCodec() {
        super("MyCodec", new Lucene80Codec());
    }

    @Override
    public DocValuesFormat docValuesFormat() {
        return myDocValuesFormat;
    }
}
```

2、自定义 DocValuesFormat 实现类
```java
public class MyDocValuesFormat extends DocValuesFormat {

    public static final String MY_EXT = "mydv";

    protected MyDocValuesFormat() {
        super("MyCodec");
    }

    @Override
    public DocValuesConsumer fieldsConsumer(SegmentWriteState state) throws IOException {
        return new MyDocValuesWriter(state, MY_EXT);
    }

    @Override
    public DocValuesProducer fieldsProducer(SegmentReadState state) throws IOException {
        return new MyDocValuesReader(state, MY_EXT);
    }
}
```

3、扩展 DocValuesConsumer 实现数据写入文件（比如通过 addBinaryField() 写 BINARY 类型字段数据）
```java
public class MyDocValuesWriter extends DocValuesConsumer {

    IndexOutput data;

    public MyDocValuesWriter(SegmentWriteState state, String ext) throws IOException {
        System.out.println("WRITE: " + IndexFileNames.segmentFileName(state.segmentInfo.name, state.segmentSuffix, ext) + " " + state.segmentInfo.maxDoc() + " docs");
        data = state.directory.createOutput(IndexFileNames.segmentFileName(state.segmentInfo.name, state.segmentSuffix, ext), state.context);
    }

    @Override
    public void addBinaryField(FieldInfo field, DocValuesProducer valuesProducer) throws IOException {
        // write data
    }

    …
}
```

4、扩展 DocValuesProducer 实现读取文件数据（比如通过 getBinary() 读 BINARY 类型字段数据）
```java
public class MyDocValuesReader extends DocValuesProducer {

    final IndexInput data;
    final int maxDoc;

    public MyDocValuesReader(SegmentReadState state, String ext) throws IOException {
        System.out.println("dir=" + state.directory + " seg=" + state.segmentInfo.name + " file=" + IndexFileNames.segmentFileName(state.segmentInfo.name, state.segmentSuffix, ext));
        data = state.directory.openInput(IndexFileNames.segmentFileName(state.segmentInfo.name, state.segmentSuffix, ext), state.context);
        maxDoc = state.segmentInfo.maxDoc();
    }

    @Override
    public BinaryDocValues getBinary(FieldInfo field) throws IOException {

        return new BinaryDocValues() { … }

    }

    …
}
```

**参考资料**  
[1]. https://lucene.apache.org/core/8_5_0/core/org/apache/lucene/codecs/package-summary.html  
[2]. What is an Apache Lucene Codec? https://www.elastic.co/cn/blog/what-is-an-apache-lucene-codec  
[3]. Build Your Own Lucene Codec! https://opensourceconnections.com/blog/2013/06/05/build-your-own-lucene-codec/
