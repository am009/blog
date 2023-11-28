---
title: Maven使用本地JAR文件
date: 2023/11/28 11:11:12
categories:
- Dev
tags:
- Java
---

Maven，如果想要修改底层依赖的库，先clone下来修改构建，然后使用本地JAR文件。StackOverflow的高赞回答里有一些坑。记录一下。

<!-- more -->

### 背景

Soot加载新的APK总是报错，而且报错了就停了。能不能让它忽略当前报错，直接放弃当前method body呢？基于这个想法我开始修改soot源码，最后的方案在[这个commit](https://github.com/am009/soot/commit/d3934e6c39a6e2993b02c7f2793d59d26c49afe3)。

## 遇到的问题

### 编译时跳过测试

修改了Soot代码，直接编译会说一些测试异常。使用`mvn -Dmaven.test.skip=true package`在打包时跳过异常。

### maven引用本地jar文件

- **基于scope=system的方式**

网上搜索首先会看到[这个回答](https://stackoverflow.com/a/22300875)，它推荐使用system类型的依赖：

```xml
<dependency>
    <groupId>com.sample</groupId>
    <artifactId>sample</artifactId>
    <version>1.0</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/src/main/resources/Name_Your_JAR.jar</systemPath>
</dependency>
```

然而，再看看就发现很多地方说这种方式不好，比如已经depricated了。直接劝退我的，是我用的shaded插件（和依赖一起打包为一个jar），不会把这种依赖加进来，直接报错找不到`Soot.G`这个类。

- **基于`mvn install:install-file`的方式**

于是采用回答里的[第二种方法](https://stackoverflow.com/a/4955695)，安装到本地maven缓存。

```
mvn install:install-file \
   -Dfile=$SCRIPTPATH/sootclasses-trunk.jar \
   -DgroupId=org.soot-oss \
   -DartifactId=soot \
   -Dversion=4.5.0-SNAPSHOT1 \
   -Dpackaging=jar \
   -DgeneratePom=true
```

问题在于，这种方式引入的依赖似乎不会引入递归的依赖。比如我发现缺了很多soot的依赖。

- **基于maven-install-plugin的方式**

根据[官方文档](https://maven.apache.org/guides/mini/guide-3rd-party-jars-local.html)，如果那边jar是maven构建的，则可以使用下面这种方式直接安装到本地缓存，因为读取了jar里的`pom.xml`文件。既然都有了pom了，依赖信息也在了。但是坑点是版本要改成3.0.1以上，因为其实这个是个已知的问题，很晚才修复。[这个comment](https://stackoverflow.com/questions/27554781/maven-install-file-isnt-resolving-dependencies#comment129402640_27554971)救了我。

```
# 删除本地的缓存
# mvn dependency:purge-local-repository -DmanualInclude="org.soot-oss:soot"
mvn org.apache.maven.plugins:maven-install-plugin:3.0.1:install-file -Dfile=$SCRIPTPATH/sootclasses-trunk.jar
```

最后发现怎么还是在跑的旧代码，可能是shaded插件的问题。还是得删掉targets文件夹。

### 检查依赖问题

使用`mvn dependency:tree`可以打印树状的依赖图

### VSCode Java插件关联本地代码

根据这个[issue](https://github.com/redhat-developer/vscode-java/issues/2391)里说的，看来只能在那边创建`-src`后缀的jar文件，可能要同时放到本地maven仓库？

### 总体构建脚本

```bash
#!/bin/bash
# 获取脚本当前路径
SCRIPTPATH="$( cd -- "$(dirname "$0")" >/dev/null 2>&1 ; pwd -P )"

# 防止还是用的旧的库
rm -rf $SCRIPTPATH/target

pushd ~/soot
# 生成patch留存
git diff --cached > $SCRIPTPATH/mypatch.patch 
# 构建依赖的库的jar包
mvn -Dmaven.test.skip=true package
cp target/sootclasses-trunk.jar $SCRIPTPATH/
mvn org.apache.maven.plugins:maven-install-plugin:3.0.1:install-file -Dfile=$SCRIPTPATH/sootclasses-trunk.jar
popd

# 构建真正的项目
mvn package
```
