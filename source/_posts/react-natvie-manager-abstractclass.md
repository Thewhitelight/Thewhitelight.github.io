---
title: ReactNative 中使用 @ReactMethod 和 @ReactProp 应注意的问题
date: 2019-10-14 15:43:46
tags:
    - Android
    - React Native
categories: 技术
---
#### 问题
公司项目中使用 React Native 实现部分功能,因为业务需求问题,比如 A 和 B 分别继承 ReactContextBaseJavaModule 且其 ReactMethod 所注解的方法也相同,但是其实现略有不同,这时候正常做法是会写抽象类或者说接口,这时候 ReactMethod 应该写在那里呢,如果在父类或者接口里,运行起来会发现找不到这个方法,如果写在实现类里会很麻烦.
<!-- more -->
#### 解决方案
##### 实现类的方法里加 ReactMethod
这种方式就很麻烦不够优雅,但是可以立马解决问题.
##### 抽象类实现或者接口继承 ReactModuleWithSpec
为什么直接在父类给方法加注解就不行呢,还是得从源码里找答案,直接搜索源码里处理注解的地方,如下所示:
``` java JavaModuleWrapper.java
@DoNotStrip
  private void findMethods() {
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "findMethods");
    Set<String> methodNames = new HashSet<>();

    Class<? extends NativeModule> classForMethods = mModuleClass;
    //获取父类
    Class<? extends NativeModule> superClass =
        (Class<? extends NativeModule>) mModuleClass.getSuperclass();
    if (ReactModuleWithSpec.class.isAssignableFrom(superClass)) {
        //如果父类实现 ReactModuleWithSpec后,会获取父类的方法
      classForMethods = superClass;
    }
    //如果父类没有实现 ReactModuleWithSpec,则会拿子类的方法,如果子类方法没有 ReactMethod 注解,则不会调用,这就解释了我们刚开始遇到的问题.
    Method[] targetMethods = classForMethods.getDeclaredMethods();

    for (Method targetMethod : targetMethods) {
      ReactMethod annotation = targetMethod.getAnnotation(ReactMethod.class);
      if (annotation != null) {
        //··· 忽略非重点代码
      }
    }
    Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
  }
```
根据源码,我们就很清楚的理解所遇到的问题了.**但是如果我们在实现类里做扩展,再加含有 ReactMethod 注解的方法时,我们运行时也会找不到此方法,所以使用时需要注意这一点**.

#### ReactProp 类似问题

当我们在抽象类中对方法使用 ReactProp 注解时,运行时可以发现,他们时生效的.这个通过查看源码发现,其递归获取了父类,然后获取了方法.
``` java ViewManagersPropertyCache.java
static Map<String, PropSetter> getNativePropSettersForViewManagerClass(
      Class<? extends ViewManager> cls) {
    if (cls == ViewManager.class) {
      return EMPTY_PROPS_MAP;
    }
    Map<String, PropSetter> props = CLASS_PROPS_CACHE.get(cls);
    if (props != null) {
      return props;
    }
    //递归获取继承了 ViewManager 的类
    props = new HashMap<>(
        getNativePropSettersForViewManagerClass(
            (Class<? extends ViewManager>) cls.getSuperclass()));
    extractPropSettersFromViewManagerClassDefinition(cls, props);
    CLASS_PROPS_CACHE.put(cls, props);
    return props;
    }
  private static void extractPropSettersFromViewManagerClassDefinition(
      Class<? extends ViewManager> cls,
      Map<String, PropSetter> props) {
    //获取这个类的public,protroct,private方法,但不包括实现接口和父类的方法
    Method[] declaredMethods = cls.getDeclaredMethods();
    //···省略无关代码
    }
```
通过 extractPropSettersFromViewManagerClassDefinition 这个方法,我们可以看出如果在**接口中对方法使用 ReactProp 时会不生效**.

遇到问题多看看源码实现,也许就会找到思路.