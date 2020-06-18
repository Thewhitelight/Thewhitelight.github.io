---
title: Maven 上传下载操作
date: 2020-03-09 19:33:20
tags:
    - Maven
categories:
    - 技术
---
Java 开发中使用最多的构建工具就是 Maven，Android 的 Gradle 构建工具使用的版本仓库也是 Maven，对于使用脚本打包出产物上传至远程仓库的方式已经很熟悉，但是有时候需要从 Maven 下载产物或者直接上传产物。如果每次都在 Maven 的服务端页面操作就很麻烦。所以如果使用脚本就很方便。
<!-- more -->

# 配置 mvn

## 安装 mvn 命令行，mac 下可以使用 `brew install mvn`。
## 配置 mvn 文件
当 mvn 安装好后命令行执行`which mvn`查看安装路径，如`/usr/local/apache-maven-3.6.3/bin/mvn`，然后`cd apache-maven-3.6.3/conf/`可以在这个目录下发现`settings.xml`，打开 xml 文件配置如下
``` xml 
<servers>
    <!-- 添加 server 节点，id 可以任意写，注意后面的命令行参数会用到，username password 为私有 maven 配置 -->
      <server>
      <id>releases</id>
      <username>***</username>
      <password>***</password>
      </server>
<servers> 
```

# 命令行上传产物
以上传`com.android.support:appcompat-v7:28.0.0`为例,比如在当前目录下有`appcompat-v7-28.0.0.aar`这个包
使用 下面命令行即可上传
`mvn deploy:deploy-file -Dfile=appcompat-v7-28.0.0.aar -DgroupId=com.android.support -DartifactId=appcompat-v7 -Dversion=28.0.0 -Dpackaging=aar -Durl=https://www.***.com/repository/maven-releases/ -DrepositoryId=releases -DpomFile=appcompat-v7-28.0.0.pom`

`-DrepositoryId`即为`xml`中配置的 id，名字必须相同
`-Dpackaging`即为上传产物类型
`-Durl`即为上传地址
`-DpomFile`即为上传的 pom 文件，可选参数

# 下载产物
## mvn install
如果有 pom 文件 直接在目录下执行 `mvn install` 就可以下载，但是前提得有 pom 较为麻烦
## 脚本下载
模拟 support 包的下载路径是这样`https://www.***.com/repository/maven-releases/com/android/support/appcompat-v7/28.0.0/appcompat-v7-28.0.0.pom`，观察 url 我们就可以发现是通过依赖包名就可以构造出来的路径，`aar、jar、sourec.jar`都是类似的路径。我们就可以通过`curl`命令构造路径直接下载到想要的产物，由于脚本过于简单就不做演示。

# 批量获取 maven 产物
由于业务需要查看 maven 里产物的内容，但是一个一个下载查看就很麻烦，所以抓包看了下 maven 搜索页面发现这个链接`https://www.***.com/service/extdirect`,在加上抓取的 Head 我们就可以模拟出一个正常的请求从而获取到想要的所有的产物链接，也可以通过开放 API(https://www.***.com/service/rest/v1/search) 获取全部产物，通过解析`json`进一步获取下载地址，通过`curl`即可下载到产物，最后通过`unzip`就可以拆包解析产物。所以流程有点长但是一次写好脚本以后，就可以很方便的重复使用。

# 最后
mvn 自己不算了解，在工作实践中慢慢的了解，多次尝试总结出一点使用方法。