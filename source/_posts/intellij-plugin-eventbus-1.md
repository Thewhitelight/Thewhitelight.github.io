---
title: Intellij Plugin 开发(一)
date: 2019-05-29 20:46:36
tags:
    Intellij Plugin
categories:
    技术
---
Intellij 有很多优秀的开源插件,让我们更加高效的编程.EventBus 虽然可以很好的解耦，但是对于使用者来说不那么方便找到发射点和接收点.EventBus Plugin 帮我们愉快的解决了这个痛点.但是由于我们项目自己封装 EventBus,所以其插件无法生效,只能通过站在巨人的肩膀上,修改轮子让他重新跑起来.

### 配置开发环境 新建项目
 开发插件可以使用收费旗舰版也可是使用免费社区版本.
 1. File->New->Project 选择 Intellij Platform Plugin 如图
 ![](https://def-201655.cos.ap-shanghai.myqcloud.com/intellij-plguin-new.png)
 Project SDK 选择 Intellij 的安装目录,这个 SDK 版本觉得这最低兼容版本,如果要适配低版本的 Intellij 或者 Android Stuido
 如果想使用 Gradle Kotlin 之类的开发方式可以选择 Gradle 然后勾选 Java、Kotlin、Intellij Platform Plugin
 ![](https://def-201655.cos.ap-shanghai.myqcloud.com/intellij-plugin-new2.png)
 2. 输入项目名称和路径,点击 Finish 即可创建插件项目
 3. 配置 SandBox
 IntelliJ Plugin Run/Debug模式运行在 SandBox 中进行的,和当前 Intellij 没什么关系,需要在 Project Structure 中设置 Sandbox Home 路径.
![](https://def-201655.cos.ap-shanghai.myqcloud.com/intellij-plugin-sandbox.png)
<!-- more -->

### 配置 plugin.xml
plugin.xml 就像是我们开发 Android 时的 AndroidMainfest.xml 文件,使用到的组件都必须在这里注册,否则就不会生效.
`<idea-version since-build="173.0"/>`这个表示插件支持的起始版本,根据[文档]( http://www.jetbrains.org/intellij/sdk/docs/basics/getting_started/build_number_ranges.html)可知是 2017.3 版本.
`<depends>com.intellij.modules.lang</depends>`是项目中用到的依赖插件
``` xml plugin.xml
<extensions defaultExtensionNs="com.intellij">
        <!-- Add your extensions here -->
        <codeInsight.lineMarkerProvider language="JAVA"
                                        implementationClass="com.***.eventbus.EventBusLineMarkerProvider"/>
</extensions>
```
extensions 定义插件的扩展类型,defaultExtensionNs 定义此项目是基于 Intellij Core 编写的插件,codeInsight.lineMarkerProvider 是一种扩展类型意为标记提示器,其实现的功能就是我们此次需要的,就像我们实现接口的方法后,实现类的方法前的图标,点击后会跳转到接口中.
经过这两个步骤就新建配置好了插件项目,我们就可以基于此实现我们的逻辑了.

### 实现 LineMarkerProvider
``` java LineMarkerProvider.java
public class EventBusLineMarkerProvider implements LineMarkerProvider {

    //图片资源应在 resources/icons 目录
    private static final Icon ICON = IconLoader.getIcon("/icons/icon.png");
    //最多查找应用数量 100 个
    private static final int MAX_USAGES = 100;

    private static GutterIconNavigationHandler<PsiElement> SHOW_SENDERS =
            (e, psiElement) -> {
                if (psiElement instanceof PsiMethod) {
                    Project project = psiElement.getProject();
                    JavaPsiFacade javaPsiFacade = JavaPsiFacade.getInstance(project);
                    //查找 project 范围内 EventBus 的PsiClass
                    PsiClass eventBusClass = javaPsiFacade.findClass("com.***.event.EventBus", GlobalSearchScope.allScope(project));
                    if (eventBusClass != null) {
                        //符合 EventBus 中 post() 方法
                        PsiMethod postMethod = eventBusClass.findMethodsByName("post", false)[0];
                        PsiMethod method = (PsiMethod) psiElement;

                        PsiClass eventClass = ((PsiClassType) method.getParameterList().getParameters()[0].getTypeElement().getType()).resolve();
                        ShowUsagesAction action = new ShowUsagesAction(new SenderFilter(eventClass));
                        //触发查找动作,弹框展示已经查找到的引用点
                        action.startFindUsages(postMethod, new RelativePoint(e), PsiUtilBase.findEditor(psiElement), MAX_USAGES);
                    }
                }
            };

    private static GutterIconNavigationHandler<PsiElement> SHOW_RECEIVERS =
            (e, psiElement) -> {
                if (psiElement instanceof PsiMethodCallExpression) {
                    PsiMethodCallExpression expression = (PsiMethodCallExpression) psiElement;
                    PsiType[] expressionTypes = expression.getArgumentList().getExpressionTypes();
                    if (expressionTypes.length > 0) {
                        PsiClass eventClass = PsiUtils.getClass(expressionTypes[0], psiElement);
                        if (eventClass != null) {
                            List<PsiElement> generatedDeclarations = new ArrayList<>();
                            generatedDeclarations.add(PsiUtils.getClass(expressionTypes[0]));
                            //找到接收点接收对象的父类,这样通过发射点发射其子类,也可以看到其父类的接收点位置
                            //例如 Event2 extend Event ,EventBus.post(new Event2()) 那么可以在发射点同时看到 Event2 和 Event 的接收点位置
                            for (PsiType superType : expressionTypes[0].getSuperTypes()) {
                                generatedDeclarations.add(PsiUtils.getClass(superType));
                            }
                            new ShowUsagesAction(new ReceiverFilter()).startFindUsages(generatedDeclarations, eventClass, new RelativePoint(e), PsiUtilBase.findEditor(psiElement), MAX_USAGES);
                        }
                    }
                }
            };

    @Nullable
    @Override
    public LineMarkerInfo getLineMarkerInfo(@NotNull PsiElement psiElement) {
        if (PsiUtils.isEventBusPost(psiElement)) {
            //判断是否是发射点
            return new LineMarkerInfo<>(psiElement, psiElement.getTextRange(), ICON, Pass.UPDATE_ALL, null, SHOW_RECEIVERS, GutterIconRenderer.Alignment.LEFT);
        } else if (PsiUtils.isEventBusReceiver(psiElement)) {
            //判断是否是接收点
            return new LineMarkerInfo<>(psiElement, psiElement.getTextRange(), ICON, Pass.UPDATE_ALL, null, SHOW_SENDERS, GutterIconRenderer.Alignment.LEFT);
        }
        return null;
    }

    @Override
    public void collectSlowLineMarkers(@NotNull List<PsiElement> elements, @NotNull Collection<LineMarkerInfo> result) {

    }

}
```
LineMarkerProvider 会在我们每次打开文件时执行,当 getLineMarkerInfo 执行后一行代码如果符合我们的条件时,则行号后会出现 ICON 定义的代码,这个icon 必须是 `16 * 16` 或者 `32 * 32` 尺寸的图标.
PSI (Program Structure Interface)程序结构接口,可以描述文件的层次结构(所谓的PSI树),这样我们就可以针对代码,逐个进行匹配分析.这个相关文档很少,但是 Intellij 自带工具可以帮我们分析文件的 psi 树,点击 Tool->View PSI Structure of Current File 这样我们就可以直观的预览需要处理的文件.PSI 对于代码所有的关键字和标点都有定义,如下图里可以清楚的看出
![](https://def-201655.cos.ap-shanghai.myqcloud.com/intellij-plugin-psi.png)
通过这个工具和结合[PsiElement 文档](http://www.jetbrains.org/intellij/sdk/docs/basics/architectural_overview/psi_elements.html)可以学习怎么使用它.

### 小结
这篇只是介绍下怎么入门,简单的了解下怎么新建项目和基础的代码实现,由于相关文档比较少,如果想要编写实用的插件,就需要参考 GitHub 中大量的插件工程,通过学习可以更快入门.
 