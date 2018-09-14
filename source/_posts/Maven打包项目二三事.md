---
title: Maven打包项目二三事
date: 2018-09-14 16:06:00
categories: web
tags:
- maven
- springboot
---

## 使用 Maven 打包项目
我们在 Idea 构建 Maven 工程时，在 Idea 的侧边栏可以调出 Maven 相关的操作选项，可以很容易的来执行生命周期，clean 可以清理编译出来的文件，compile 对文件进行编译，package 将项目进行打包，install 将打包完成的包发布到本地的 Maven 仓库。如果我们没有为各个生命周期配置插件，Maven 将使用默认的插件完成这些生命周期。例如，Maven 默认使用 maven-jar-plugin 对项目进行打包，打包生成的包只能作为一个依赖包使用，不能作为一个可执行包，要作为一个可执行包，需要配置 `Main-Class`
<!-- more -->

## Maven 项目读取本地 Jar 包
要在本地项目中引入本地的 Jar 包，网上很多，这里就说说通过 Scope 引入本地包的方法。
在 pom.xml 中添加下面的 dependency
```
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-lang3</artifactId>
    <version>3.8</version>
    <scope>system</scope>
    <systemPath>${basedir}/lib/commons-lang3-3.8.jar</systemPath>
</dependency>
```
在项目中则需要在根目录下新建 lib 文件夹，然后将 jar 包置入

其实如果不想后续打包可能造成的麻烦，最好是 install 到本地仓库然后引入，只不过最近在客户现场做项目的时候，电脑都装不了 maven，我还是用 idea 自带的 maven 的

## 使用 maven-jar-plugin 打包可执行文件
使用 maven-jar-plugin 打包出来的可执行 jar 是不包含依赖的，所以我们需要将依赖也打包出来，需要使用 dependency-plguin 的配置
```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.0.2</version>
    <configuration>
        <archive>
            <manifest>
                <addClasspath>true</addClasspath>
                <classpathPrefix>lib/</classpathPrefix>
                <mainClass>Test</mainClass>
            </manifest>
            <!--可以把依赖本地系统的Jar包加入Manifest文件中-->
            <manifestEntries>
                <Class-Path>lib/commons-lang3-3.8.jar</Class-Path>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>

<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-dependency-plugin</artifactId>
    <version>3.1.1</version>
    <executions>
        <execution>
            <id>copy-dependencies</id>
            <phase>package</phase>
            <goals>
                <goal>copy-dependencies</goal>
            </goals>
            <configuration>
                <outputDirectory>${project.build.directory}/lib</outputDirectory>
                <overWriteReleases>false</overWriteReleases>
                <overWriteSnapshots>false</overWriteSnapshots>
                <overWriteIfNewer>true</overWriteIfNewer>
            </configuration>
        </execution>
    </executions>
</plugin>	
```
jar-plugin 的 <classpathPrefix> 指定了 classpath 的位置，本案例中即读取可执行包同级目录下的 lib 目录，可执行 jar 的 Manifest 文件如下。dependency-plugin 则将 maven 中的依赖包打包到 <outputDirectory> 指定目录下。
```
Manifest-Version: 1.0
Built-By: akis
Class-Path: lib/guava-26.0-jre.jar lib/jsr305-3.0.2.jar lib/checker-qu
 al-2.5.2.jar lib/error_prone_annotations-2.1.3.jar lib/j2objc-annotat
 ions-1.1.jar lib/animal-sniffer-annotations-1.14.jar lib/commons-lang
 3-3.8.jar
Created-By: Apache Maven 3.3.9
Build-Jdk: 1.8.0_172
Main-Class: Test
```

### 将依赖包打包到可执行包中
要将依赖包中打包到可执行包中，有很多插件能做到，这里介绍一下 shade-plugin
```
<plugin>
<groupId>org.apache.maven.plugins</groupId>
<artifactId>maven-shade-plugin</artifactId>
<version>3.1.1</version>
<executions>
  <execution>
    <goals>
      <goal>shade</goal>
    </goals>
    <configuration>
      <transformers>
        <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
          <manifestEntries>
            <Main-Class>${app.main.class}</Main-Class>
            <X-Compile-Source-JDK>${maven.compiler.source}</X-Compile-Source-JDK>
            <X-Compile-Target-JDK>${maven.compiler.target}</X-Compile-Target-JDK>
          </manifestEntries>
        </transformer>
      </transformers>
    </configuration>
  </execution>
</executions>
</plugin>	
```
该插件能将依赖包导入可行性 jar 包，但是不会引入上文提到的本地引入的依赖包，如果想导入本地依赖的包到可执行 jar，可以使用 [addjars-maven-plugin](https://blog.csdn.net/gaopu12345/article/details/78596830)，有兴趣的读者可以试试

当然使用 java -cp 命令而不是 java -jar 命令来执行主类的话，可以暂时解决外置依赖没打包进依赖包的问题
```
// java -cp xx.jar:lib/xx.jar packageName.MainClassName
java -cp testpackage-1.0-SNAPSHOT.jar:lib/commons-lang3-3.8.jar Test
```

## springboot 项目打包
springboot 项目默认使用了 spring-boot-maven-plugin，默认会打包一个嵌入了 tomcat 的可执行 jar 包，最近做的一个项目里就需要引入外置的 jar 包，并且不将其打包到 springboot 的可执行 jar 里，所以可以用上文提到的引入方法。这样如果我们的外置 jar 更新了，只需要替换掉外置的这个 jar，已经打包的应用则不需要更改。
那么需要解决的问题是在运行 springboot 打包出来的可执行 jar 的时候能够读到外置的这个 jar 依赖。

首先给出解决方案，配置 spring-boot-maven-plugin，将其 layout 改为 ZIP
```
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <layout>ZIP</layout>
                    <executable>true</executable>
                </configuration>
            </plugin>
```
然后在运行的时候加上 -Dloader.path 这个参数，lib 是 springboot 应用同级目录下的外置依赖目录，我们的外置依赖就放在这个 lib 中
```
java -Dloader.path=lib ph_pg-0.0.1-SNAPSHOT.jar --spring.profiles.active=prod
```
原因是 springboot 应用启动时时从 org.springframework.boot.loader.JarLauncher 开始启动的，该启动器会忽略外置的 classpath 下的包，因此需要替换成 PropertiesLauncher，配置 layout 为 ZIP 后，springboot 就从 PropertiesLauncher 开始启动，并且使用 loader.path 配置外置的 classpath
### 参考：
[StackOverflow](https://stackoverflow.com/questions/40994851/spring-boot-executable-jar-layout-changes-break-classpath)
[Springboot Reference](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#executable-jar-launching)
[SpringBoot加载外部依赖](https://www.jianshu.com/p/a2cf2336a48c)
[Springboot的“fat jar” 变成 “thin jar”](https://blog.csdn.net/timedifier2/article/details/53925825), 这篇很值得看下

## Maven 打包时排除掉某个文件
项目里 License 模块，LicenseGenerator 也在模块里，但是打包的时候我们就不能把这个文件打包到依赖包里了，解决方法是，打包时，让这个文件不参与编译就行了，如下，需要注意的是其他的 java 类里也不要 import 这个 LicenseGenerator，不然还是会被编译。我们及时清除不需要的 import 是个好习惯
```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>1.8</source>
        <target>1.8</target>
        <excludes>
            <exclude>path/to/license/LicenseGenerator.java</exclude>
        </excludes>
    </configuration>
</plugin>
```

## 小结
项目外包不是个好差事，特别是当这个项目小到一个人能完成时，这意味着杂事多，挑战小。闲来无事，记录下碰到的 Maven 打包的问题吧



