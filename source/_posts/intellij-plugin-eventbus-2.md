---
title: Intellij Plugin 开发(二)
date: 2019-07-10 22:17:55
tags:
    Intellij Plugin
categories:
    技术
---
上一篇介绍怎样新建一个插件工程,本篇分析 EventBus 插件的实现.
此插件涉及较多的 Java GUI 代码,但是自己也不是很熟悉,只能做大概分析,重点在于分析扫描引用点.
### 扫描规则
我们项目对于 EventBus 有特殊的封装,就需要插件可以扫描出`XXX.send(new NotifyEvent())`(XXX 类也可能会有子类),可以找出这个 Event 所有的接收点,并且对于继承了 Sender 的类页都应该支持.
``` java XXXX.java
    public static void notify(){
        send(new NotifyEvent());
    }
    static void send(Object eventModel) {
          Smart.getEventBus().post(eventModel);
      }
```
<!-- more -->
#### 判断发射点
当我们打开一个 java 文件时,就会框架就会调用 getLineMarkerInfo 方法去扫描当前类的每一行代码(这么说不恰当,应该是每个字段也包括空格、空行之类,所有的信息都会转化成 PsiElement),然后我们通过 PsiElement 去判断是否时符合我们的规则.例如判断 EventBus 发射点.
``` java PsiUtil.java
   public static boolean isEventBusPost(PsiElement psiElement) {
        //判断 psiElement 是否是方法(包括构造方法)的调用
        if (psiElement instanceof PsiCallExpression) {
            PsiCallExpression callExpression = (PsiCallExpression) psiElement;
            PsiMethod method = callExpression.resolveMethod();
            if (method != null) {
                String name = method.getName();
                PsiElement parent = method.getParent();
                //判断方法名是否是发射点 我们的目标是 Smart.getEventBus().post() 和 XXXX.send() 和一些特殊形式
                if ("post".equals(name) && parent instanceof PsiClass) {
                    PsiClass implClass = (PsiClass) parent;
                    return isEventBusClass(implClass) || isSuperClassEventBus(implClass);
                } else if ("send".equals(name) && parent instanceof PsiClass) {
                    PsiClass implClass = (PsiClass) parent;
                    return isEventSenderClass(implClass) || isSuperClassEventSender(implClass);
                }
            }
        }
        return false;
    }

    //判断类是否是我们自封装的类
    private static boolean isEventBusClass(PsiClass psiClass) {
        return "SmartEventBus".equals(psiClass.getName());
    }

    //判断类是否是我们自封装类的子类
    private static boolean isSuperClassEventBus(PsiClass psiClass) {
        PsiClass[] supers = psiClass.getSupers();
        if (supers.length == 0) {
            return false;
        }
        for (PsiClass superClass : supers) {
            if ("SmartEventBus".equals(superClass.getName())) {
                return true;
            }
        }
        return false;
    }
```
#### 判断接收点
判断符合我们规则的接收点
``` java
    public static boolean isEventBusReceiver(PsiElement psiElement) {
        if (psiElement instanceof PsiMethod) {
            PsiMethod method = (PsiMethod) psiElement;
            String name = method.getName();
            //支持 EventBus 2.0 版本的 onEvent 和 onEventMainThread 两种接收点,且只有一个参数并且是 PsiClassType 类型,不至于造成对其他方法的误判
            return ("onEvent".equals(name) || "onEventMainThread".equals(name)) && method.getParameterList().getParametersCount() == 1 && method.getParameterList().getParameters()[0].getType() instanceof PsiClassType;
        }
        return false;
    }
```
以上就是针对我们项目关于扫描的自定义代码,通过这几个简单的规则就可以做到预期的效果.
### 查找使用
`ShowUsagesAction`继承`AnAction`当我们点击 icon 后会执行`ShowUsagesAction.actionPerformed()`方法,在此处执行搜索全局代码、收集使用点、显示搜索框和提示框等,需要深入了解 intellij 源码.由于能力有限不敢做过多分析,具体代码可以查看[ShowUsagesAction源码](https://github.com/kgmyshin/eventbus-intellij-plugin/blob/master/src/com/kgmyshin/ideaplugin/eventbus/ShowUsagesAction.java).由于源码几乎没有注释,所以阅读起来有一定困难.

### 参考
[IntelliJ Platform SDK](http://www.jetbrains.org/intellij/sdk/docs/welcome.html)
[eventbus-intellij-plugin](https://github.com/kgmyshin/eventbus-intellij-plugin)

