---
title: 用 BECAUSE 驾驭你的 Gradle 依赖 (译)
date: 2020-02-20 23:39:34
tags:
    - Gradle
    - 译文
categories:
    - 技术
---
# 译文
你是否知道可以指定使用某个依赖或者某个版本的依赖的原因.
是的,在[API](https://docs.gradle.org/current/javadoc/org/gradle/api/artifacts/Dependency.html#because-java.lang.String-)和[DOCS](https://docs.gradle.org/current/userguide/declaring_dependencies.html#sec:documenting-dependencies)中
{% note info %}
void because?(@Nullable String reason)

Sets the reason why this dependency should be used.
Since:
4.6
{% endnote %}
<!-- more -->
基本用法如下所示
``` groovy
implementation("com.blundell:app-auth android:1.2.3") {
    because "We use fragment auth, original AppAuth library does not support this feature."
}
```
Gradle 允许你写一两句为什么选择改依赖的句子，并且可以使用以下命令提取该注释，以显示在报告中：
`gradle -q dependencyInsight –dependency app-auth android`
但是这不是我有兴趣使用它的原因！
此功能最主要的收益就是使用在敏捷团队中，总有一些使用某种依赖的原因，有时你会坚持使用早前版本以实现兼容性。有时你因为编译时间和没有反射需要使用一个解析库里而不是另一个。有时你需要支持多个国家并且是唯一的依赖，即使他不是标准的做法。
所有这个原因通常都会消失在时间的迷雾中，如果幸运的话，它们会出现在 git commit message 中（但是说实话，它们不会首先被检查）。
使用 because 函数可以使你对特定的业务决策进行说明，并想团队其他成员明确说明为啥这么做。 与希望/查找/忽略和更改依赖项的人一起节省时间，精力和潜在的错误。
所以开始使用它！ 🙂
``` groovy
implementation("com.blundell:android-blog:1.2.3") {
    because "If we didn't Blundell might cry."
}
```
# 原文
[原文地址](https://blog.blundellapps.co.uk/tame-your-gradle-dependencies-just-because/3)