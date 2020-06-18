---
title: AndroidX 迁移脚本实现
date: 2019-12-07 10:59:16
tags:
  - Git
  - Shell
categories:
  - 技术
---
# 需求
现在 Android 10 早已经发布,当 targetVersion 为29时必须使用 AndroidX 并且现在新建项目时已经必须勾选 AndriodX 选项了,所以需要开始迁移工作了,有些三方库也只提供 AndroidX 版本,我们遇到 QQ 音乐和 GoogleMap 的 SDK 就只提供 AndroidX 版本.如果对于单个工程而言只需要 Refactor->Migrate to AndroiX 即可以完成自动迁移,但是对于有更多项目而言使用这样的方式虽然可以完成迁移,但是太过于繁琐,也不好统一 AndroidX 库的版本.为了方便整个团队项目进行迁移,所以使用脚本进行迁移会方便很多.
# 解决方案
整个流程很简单
1. checkout 新分支
2. 升级新 gradle 版本
3. 升级 gradle tool 版本
4. gradle.properties 添加 AndroidX 支持
5. 替换工程里所有 support 包名
6. 替换工程里所有 support 依赖
7. 添加混淆文件

梳理完成流程,就可以动手迁移了.需要注意 AndroidX 必须使用 gradle tool 3.2 版本以上,为了保证兼容性,工程当前最好使用 supprot 28 的依赖.
<!-- more -->
## checkout 迁移新分支
``` git
$(cd .. | git branch androidx_migrate)
$(cd .. | git checkout androidx_migrate)
```
## 升级 gradle 版本
将 gradle 升级至最新.`>/dev/null`相当于捕获异常信息.
`./gradlew wrapper --gradle-version 5.4.1 >/dev/null`
## 升级 gradle tools
``` shell
$(sed -i "" "s/com.android.tools.build:gradle:.*/com.android.tools.build:gradle:3.5.2/g" ${BUILD_FILE})
```
`${BUILD_FILE}`为根目录的`build.gradle`,替换掉老版本.注意这里的 gradle tool 应该与 gradle 版本对应.
## 修改 gradle.properties
``` shell
echo '\nandroid.useAndroidX=true
android.enableJetifier=true' >>$PROJECT_DIR/gradle.properties
```
`android.useAndroidX=true`Android 插件会使用对应的 AndroidX 库（而非支持库）
`android.enableJetifier=true`Android 插件会通过重写其二进制文件来自动迁移现有的第三方库以使用 AndroidX
## 替换 support 包
首先应该找到 support 与 androidx 的对应关系,不过 google 已经帮我们整理好这份[文档](https://developer.android.com/jetpack/androidx/migrate/class-mappings),通过关系命令行替换掉相应的包名就可以了
``` shell
from_pkgs=""
replace=""
while IFS=, read -r from_pkg to_pkg
do
    from_pkgs+=" -e $from_pkg"
    replace+="; s/$from_pkg/$to_pkg/g"
done <<< "$(tail -n +2 $MAPPING_FILE)"

rg --files-with-matches -t java -t kotlin -t xml -F $from_pkgs $PROJECT_DIR | xargs perl -pi -e "$replace"
```
`$MAPPING_FILE)`为 androidx_class_map.csv 的目录,这个文件就是 gooogle 提供的文档,但是用官方的文件出现了替换后对应不正确的问题,我自己改变下文件里的顺序后,则可以正常使用了.
`$(tail -n +2 $MAPPING_FILE)`相当于`cat`只是从第2行读取到文件结尾.因为第1行非 support 对应关系.
`replace+="; s/$from_pkg/$to_pkg/g"`是用来进行替换的规则,使用过`sed`就可以理解这个操作.意为将本行里的`$to_pkg`替换为`$from_pkg`,当有多个内容替换时用`;`分隔.
`rg`是比`find`更快的搜索工具.通过`rg`搜索到我们需要替换的文件类型,它内置了很多文件类型,文末参考里有详细教程.`xml`里的布局也需要替换掉,所以也需要把包含 support 包的相关文件搜索出来.
`xargs`是给其他命令传递参数的一个过滤器，也是组合多个命令的一个工具.
`perl`的正则支持比`sed`好很多,所以使用它可以少踩很多正则上的坑.
## 替换 support 依赖
``` shell
from_pkgs=""
replace=""
while IFS=, read -r from_pkg to_pkg
do
    from_pkgs+=" -e $from_pkg"
    replace+="; s/(\"|')$from_pkg.*(\"|')/\"$to_pkg\"/g"
    #如果使用sed 正则就需要写成这种形式,太过繁琐了
    #replace+="; s/\(\\\"\|'\)$from_pkg.*\(\\\"\|'\)/\\\"$to_pkg\\\"/g"
done <<< "$(tail -n +2 $ARTIFACT_MAPPING_FILE)"

rg --files-with-matches -t groovy -F $from_pkgs $PROJECT_DIR | xargs perl -pi -e "$replace"
```
和替换 support 包的代码差不多,只是换了读取的文件,只是这里的正则稍有稍有不同.
在项目里会有各种形式的依赖写法如
```
api "com.android.support:support-v4:${rootProject.ext.supportLibVersion}"
api "com.android.support:support-v4:28.0.0"
api 'com.android.support:support-v4:28.0.0'
```
所以需要`("|')package.*("|')`匹配所有的形式去替换成我们需要的版本.
`$ARTIFACT_MAPPING_FILE`为依赖的映射文件.
``` csv
Old build artifact,AndroidX build artifact
com.android.support:support-v4:,androidx.legacy:legacy-support-v4:1.0.0
```
这里的`1.0.0`则可以写成统一的依赖版本.比如`${rootProject.ext.v4Version}`这样对于便于统一版本管理.
## 混淆文件
由于包名变化所以需要新增混淆文件如
```
-dontwarn com.google.android.material.**
-keep class com.google.android.material.** { *; }

-dontwarn androidx.**
-keep class androidx.** { *; }
-keep interface androidx.** { *; }
```

总结
至此就完成这个那个项目迁移,整个过程根据工程大小会耗时几到几十秒钟,大量的时间消耗在升级 gradle,检索替换文件耗时很少.掌握一些脚本可以极大的提高我们的开发效率.
[迁移脚本源码](https://github.com/Thewhitelight/androidx_migrate_scripts)
# 参考
[rg 教程](https://www.hi-linux.com/posts/29245.html)
[官方迁移文档](https://developer.android.com/jetpack/androidx)
[参考迁移脚本](https://gist.github.com/loganj/7535a13e98be83460f362b63dbd13e07)