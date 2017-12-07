---
title: Maven中profiles的使用
tags:
    - Maven
    - profiles
    - com.sun.xml.internal.xsom.impl.scd.Iterators
    - Java编译环境
    - Java执行环境
---

### 问题背景
项目打包正常，服务器上启动后，抛“ClassNotFoundException:com.sun.xml.internal.xsom.impl.scd.Iterators” 和 “NoClassDefFoundError: com/sun/xml/internal/xsom/impl/scd/Iterators$Map”相关的异常，但本地启动正常。

<!--more-->

### 问题分析
初步怀疑是包冲突导致，但是根据“com.sun.xml.internal.xsom.impl.scd.Iterators”始终无法定位到该类来源于哪个包，多次处理包冲突，依然无果。
在 [ssm启动问题](http://bbs.csdn.net/topics/392191673) 博文中看到该类源自于tools.jar。
在项目中查找，发现 tools.jar 是被阿里druid引入的。以下是druid的pom.xml中的一部分：
```
<profiles>
	<profile>
		<id>default-profile</id>
		<activation>
			<activeByDefault>true</activeByDefault>
			<file>
				<exists>${env.JAVA_HOME}/lib/jconsole.jar</exists>
			</file>
		</activation>
		<properties>
			<toolsjar>${env.JAVA_HOME}/lib/tools.jar</toolsjar>
			<jconsolejar>${env.JAVA_HOME}/lib/jconsole.jar</jconsolejar>
		</properties>

		<dependencies>
			<dependency>
				<groupId>com.alibaba</groupId>
				<artifactId>jconsole</artifactId>
				<version>1.6.0</version>
				<scope>system</scope>
				<systemPath>${jconsolejar}</systemPath>
				<optional>true</optional>
			</dependency>
			<dependency>
				<groupId>com.alibaba</groupId>
				<artifactId>tools</artifactId>
				<version>1.6.0</version>
				<scope>system</scope>
				<systemPath>${toolsjar}</systemPath>
				<optional>true</optional>
			</dependency>
		</dependencies>
	</profile>
	<profile>
		<id>mac-profile</id>
		<activation>
			<activeByDefault>false</activeByDefault>
			<file>
				<exists>${java.home}/../Classes/jconsole.jar</exists>
			</file>
		</activation>
		<dependencies>
			<dependency>
				<groupId>com.alibaba</groupId>
				<artifactId>jconsole</artifactId>
				<version>1.6.0</version>
				<scope>system</scope>
				<systemPath>${java.home}/../Classes/jconsole.jar</systemPath>
			</dependency>
		</dependencies>
	</profile>

	<profile>
		<id>mac-profile-oracle-jdk</id>
		<activation>
			<activeByDefault>false</activeByDefault>
			<file>
				<exists>${java.home}/../lib/jconsole.jar</exists>
			</file>
		</activation>
		<dependencies>
			<dependency>
				<groupId>com.alibaba</groupId>
				<artifactId>jconsole</artifactId>
				<version>1.8.0</version>
				<scope>system</scope>
				<systemPath>${java.home}/../lib/jconsole.jar</systemPath>
			</dependency>

			<dependency>
				<groupId>com.alibaba</groupId>
				<artifactId>tools</artifactId>
				<version>1.8.0</version>
				<scope>system</scope>
				<systemPath>${java.home}/../lib/tools.jar</systemPath>
			</dependency>
		</dependencies>
	</profile>
</profiles>
```
起初我误以为只有第一个profile才会生效，因为它设置了activeByDefault表示默认激活，而且其file判断中应该为真，所以只有第一个profile才会生效，其他不生效。其所配置的tools.jar optional为true，因而其不具备传递性，就一定不会在我的项目引用包中出现，但事实是其出现了。一个问题未解决，又出现一个新问题。网上很多地方对profiles的解释也很含糊，例如 [Maven简介（三）—— profile介绍](http://elim.iteye.com/blog/1900568)。最终找到了 [maven pom进阶教程 - profiles](https://my.oschina.net/u/2343729/blog/830924)解决了我的疑惑。以下是从这篇博文中的关键引用：
> 当需要根据不同的环境，采用不同的依赖库、配置、插件，可以使用profiles。配置要点是activation激活条件，在以下几种情况下，profile将被激活:
> 1. activeByDefault=true，并且其它的profile都未被激活，此时无视匹配条件，保底作用，保证必定有一个profile被激活。
> 2. activeByDefault=false，并且activation下的所有条件都匹配，如果activation下没有匹配条件，则视为不匹配。
> 3. mvn执行命令时，加入-P，此时必定激活id为profileID的profile，无视匹配条件。

那么druid配置中，由于第三个profile.file验证为真，实际上第一个profile和第三个profile都生效了，由于都是对tools.jar的引用控制，则第三个profile覆盖了第一个profile，从而导致tools.jar可传递，在我的项目依赖包中便出现了tools.jar。
题外话，Maven中optional的优先级高于scope，optional为true时，表示该包一定不具有传递性，scope相关的知识可以参考 [Maven实战（六）依赖](http://tangyanbo.iteye.com/blog/1503957)。此处scope=system，就是为了引入本地包，而不是maven仓库中的包设定的。
回到正题，那在项目引入druid时排除tools.jar，应该就可以解决当前的问题。配置排除后，执行编译，才发现自己有个类中误引用了com.sun.xml.internal.xsom.impl.scd.Iterators.Map，修改后，一切就都正常了。
那么之前线上无法启动但本地可以启动还是无法解释。线上打包都可以成功，说明编译期没有问题，为什么启动不成功呢，那就是执行时无法加载到tools.jar。
经过思考，maven在编译代码时，使用的是jdk中的javac，jdk中是包含tools.jar的，这样编译是没有问题的，由于我本地开发环境，使用的也是jdk中的javac编译，jdk中的java执行，因而也都没问题。但是服务器上执行java命令使用的就不是jdk中的java了，而是jre中的java，但在jre中是没有tools.jar的，所以，这就解释了之前为什么服务器上执行失败了。
![](/img/maven_profiles.png)