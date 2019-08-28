---
layout: post
title: "Elasticsearch 插件开发简明指南"
categories: blog
tags: ["Elasticsearch", "es 插件开发"]
---

下面作为一个简明指南，介绍一下开发 Elasticsearch 插件所需要的主要流程，参照流程大家即可方便的进行自定义功能插件开发。

演示环境配置及要求：
```
JDK：1.8，
IDE：IntelliJ IDEA，
Elasticsearch 版本：5.6.13，
项目管理工具：Maven
```

**第一步：创建项目骨架（skeleton）**

在 IDEA 中通过 Maven 创建一个项目，比如称为 plugin-demo，其生成的 pom.xml 如下：
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.netease.panther</groupId>
    <artifactId>plugin-demo</artifactId>
    <version>1.0-SNAPSHOT</version>

</project>
```

**第二步：添加 Elasticsearch 核心依赖**

在 pom.xml 中添加如下依赖：
```
<properties>
    <elasticsearch.version>5.6.13</elasticsearch.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.elasticsearch</groupId>
        <artifactId>elasticsearch</artifactId>
        <version>${elasticsearch.version}</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

由于插件开发完成后必须在 Elasticsearch 集群环境中运行，因此依赖中的 scope 只需要填写 provided 即可。

**第三步：插件开发配置**

根据 Elasticsearch 要求（https://www.elastic.co/guide/en/elasticsearch/plugins/master/plugin-authors.html#_plugin_descriptor_file），所有的插件必须包含一个名为 plugin-descriptor.properties 的插件描述文件，对其内容有要求且必须放置在 elasticsearch 目录下。我们在 src/main/resources 目录下创建 plugin-descriptor.properties 并添加内容如下：
```
description=A Elasticsearch demo plugin
version=0.1-beta1
name=DemoPlugin
classname=com.netease.panther.DemoPlugin
java.version=1.8
elasticsearch.version=5.6.13
```

另外，由于 Elasticsearch 要求插件需要打包成 zip 文件，我们可以配置 Maven Assembly 插件使其自动生成。创建文件 src/main/assembly/plugin.xml，添加内容如下：
```
<?xml version="1.0"?>
<assembly>
    <id>plugin</id>
    <formats>
        <format>zip</format>
    </formats>
    <includeBaseDirectory>false</includeBaseDirectory>
    <files>
        <file>
            <source>${project.basedir}/src/main/resources/plugin-descriptor.properties</source>
            <outputDirectory>elasticsearch</outputDirectory>
            <filtered>true</filtered>
        </file>
    </files>
    <dependencySets>
        <dependencySet>
            <outputDirectory>elasticsearch</outputDirectory>
            <useProjectArtifact>true</useProjectArtifact>
            <useTransitiveFiltering>true</useTransitiveFiltering>
        </dependencySet>
    </dependencySets>
</assembly>
```

配置 Maven Assembly 插件：
```
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <appendAssemblyId>false</appendAssemblyId>
                    <outputDirectory>target</outputDirectory>
                    <descriptors>
                        <descriptor>src/main/assembly/plugin.xml</descriptor>
                    </descriptors>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

在 resource 中过滤 plugin-descriptor.properties（无需打包进 plugin 的 jar 中）：
```
<resources>
            <resource>
                <directory>src/main/resources</directory>
                <filtering>false</filtering>
                <excludes>
                    <exclude>plugin-descriptor.properties</exclude>
                </excludes>
            </resource>
        </resources>
```

最后，配置 Maven 的 compile 插件使其使用 jdk 1.8 版本：
```
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.7.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
```

**第四步：创建插件项目结构及主类**

在 src/main/java 下创建 package: `com.netease.panther`，在 package 下创建主类：DemoPlugin，并继承Elasticsearch Plugin 类：
```
package com.netease.panther;

import org.elasticsearch.plugins.Plugin;

public class DemoPlugin extends Plugin {
}
```

**第五步：添加测试框架**

Elasticsearch 本身提供了一个非常方便的测试框架，便于大家进行单元测试及集成测试，详细可参考：https://www.elastic.co/guide/en/elasticsearch/reference/current/testing.html，我们将其添加至项目中。

添加测试相关依赖：

```
<properties>
    <log4j.version>2.11.1</log4j.version>
</properties>

        &lt;-- for testing -->
        <dependency>
            <groupId>org.elasticsearch.test</groupId>
            <artifactId>framework</artifactId>
            <version>${elasticsearch.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>${log4j.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>${log4j.version}</version>
            <scope>test</scope>
        </dependency>
```

继承 ESIntegTestCase 进行项目集成测试：
```
package com.netease.panther;

import org.elasticsearch.action.admin.cluster.node.info.NodeInfo;
import org.elasticsearch.action.admin.cluster.node.info.NodesInfoResponse;
import org.elasticsearch.plugins.Plugin;
import org.elasticsearch.plugins.PluginInfo;
import org.elasticsearch.test.ESIntegTestCase;

import java.util.Arrays;
import java.util.Collection;

public class DemoPluginTest extends ESIntegTestCase {

    @Override
    protected Collection<Class<? extends Plugin>> nodePlugins() {
        return Arrays.asList(DemoPlugin.class);
    }

    /**
     * Test if plugin is loaded
     */
    public void testPluginIsLoaded() {

        NodesInfoResponse response = client().admin().cluster().prepareNodesInfo().setPlugins(true).get();

        for (NodeInfo nodeInfo : response.getNodes()) {

            boolean found = false;
            for (PluginInfo pluginInfo : nodeInfo.getPlugins().getPluginInfos()) {
                if (pluginInfo.getName().equals(DemoPlugin.class.getName())) {
                    found = true;
                    break;
                }
            }

            // assert found
            assertTrue(found);
        }
    }
}
```

**第六步：编译**

```
mvn clean package
```

编译完成后，在 target 目录下即可获得 `plugin-demo-1.0-SNAPSHOT.zip` 插件文件。 

**第七步：安装**

使用 Elasticsearch 插件安装命令：

```
bin/elasticsearch-plugin install file:///path/to/target/plugin-demo-1.0-SNAPSHOT.zip
```

**写在最后**

至此，所有的流程都已经完成了，根据各自业务具体的需要，即可以在 DemoPlugin 扩展相关的接口进行具体的逻辑实现。Elasticsearch 为各个功能模块都预留了相应的扩展接口，架构设计非常优秀，have fun！

关于 IntelliJ IDEA 中编译错误的说明

在某些版本 IntelliJ IDEA 中，编译上述工程时可能会出现“jar hell”异常，出现此异常时可以参考 https://github.com/elastic/elasticsearch/blob/master/CONTRIBUTING.md#configuring-ides-and-running-tests 部分解决。

参考资料：  
[1] http://david.pilato.fr/blog/2016/10/16/creating-a-plugin-for-elasticsearch-5-dot-0-using-maven-updated-for-ga/

