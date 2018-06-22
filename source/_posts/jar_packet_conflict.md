---
title: Jar包冲突检查及处理
date: 2018-06-12 20:15
tags:
    - 包冲突
---

### 背景
项目中引入fasterxml Jar包后，出现如下图所示的Jar包冲突：
![](/img/jar_packet_conflict/packet_conflict.png)
通过项目引入的包：
![](/img/jar_packet_conflict/external_libraries.png)
在所有引入的fasterxml包中，只有jackson-annotations为2.8.0，其他均为2.8.8。修改了引入包jackson-annotations为2.8.8后，问题仍然没有解决。

<!--more-->

### 查找冲突的包
一种方法是，根据类名SerializationConfig查找该类，发现有多个包中存在该类，从而找到可能冲突的包。
另一种比较好的做法是，项目增加JVM启动参数-verbose:class，打印类加载信息，从中找到com.fasterxml.jackson.databind.SerializationConfig的相关包：
```
[Loaded com.fasterxml.jackson.databind.SerializationConfig from file:/D:/Maven/.m2/com/youzan/open-sdk-java/2.0.2/open-sdk-java-2.0.2.jar]
```
最终发现是open-sdk-java包将fasterxml 2.5版本打在了自己包内部，如下图所示：
![](/img/jar_packet_conflict/youzhan_package.png)

### 解决办法
使用JarJar工具包统一修改包中类路径。
如下pom文件：
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>com.github</groupId>
		<artifactId>mydog-parent</artifactId>
		<version>1.0.0-SNAPSHOT</version>
	</parent>
	<groupId>com.youzan</groupId>
	<artifactId>open-sdk-java</artifactId>
	<version>2.0.2-fix</version>
	<dependencies>
		<dependency>
			<groupId>com.youzan</groupId>
			<artifactId>open-sdk-java</artifactId>
			<version>2.0.2</version>
			<optional>true</optional>
		</dependency>
	</dependencies>
	<build>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
			</plugin>
			<plugin>
				<groupId>org.sonatype.plugins</groupId>
				<artifactId>jarjar-maven-plugin</artifactId>
				<executions>
					<execution>
						<phase>package</phase>
						<goals>
							<goal>jarjar</goal>
						</goals>
						<configuration>
							<includes>
								<include>com.youzan:open-sdk-java</include>
							</includes>
							<rules>
								<rule>
									<pattern>com.google.**</pattern>
									<result>youzan.com.google.@1</result>
								</rule>
								<rule>
									<pattern>com.fasterxml.jackson.**</pattern>
									<result>youzan.com.fasterxml.jackson.@1</result>
								</rule>
								<rule>
									<pattern>org.apache.**</pattern>
									<result>youzan.org.apache.@1</result>
								</rule>
								<rule>
									<pattern>org.codehaus.jackson.**</pattern>
									<result>youzan.org.codehaus.jackson.@1</result>
								</rule>
							</rules>
						</configuration>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>
</project>
```
配置一个空项目，其pom为上述内容，打包，上传到Maven库中，项目中引用改动后的新包：
```
<groupId>com.youzan</groupId>
<artifactId>open-sdk-java</artifactId>
<version>2.0.2-fix</version>
```
于是，该包结构变为：
![](/img/jar_packet_conflict/youzhan_package_fix.png)
对于FasterXML，需要做特殊处理，在META-INF services中com.fasterxml.jackson.core.JsonFactory和com.fasterxml.jackson.databind.ObjectMapper文件名及文件内容需要统一手动修改为youzan.com.fasterxml.jackson.core.JsonFactory和youzan.com.fasterxml.jackson.databind.ObjectMapper，更改该包版本，并将该包上传到maven库中，再次引入该包，就大功告成了。这里是JsonFactory和ObjectMapper实现类的配置。

### 附加说明

#### 将第三方包打入自己Jar中
可以使用Maven plugin：maven-shade-plugin来实现。