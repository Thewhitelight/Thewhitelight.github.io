---
title: 利用 AOP 对方法添加监控日志
date: 2019-01-02 11:34:18
tags:
    - Android
    - AspectJ
    - Javassist
categories: 
    - 技术
---
在查看同事写的代码时,发现各种方法调用有点混乱,不易快速梳理逻辑.所以想着是否可以通过打印方法日志,从而观察各种方法间的调用逻辑.
#### 需求

1. 打印方法名称、入参、耗时、返回值
2. 对项目所有方法生效
3. 可以自定义排除不打印的类
4. 可作为三方库
<!-- more -->

#### 实现
本项目使用 Javassist 实现了以下需求.
##### AspectJ
这个功能让我想起了 JakeWharton 大神的 hugo 库,通过 AspectJ 对添加注解的方法插入 Log 在调用的时候就可以观察调用顺序及参数.但是这个库不能满足所有方法的打印,只能对添加了注解的方法打印,手动给方法添加注解工程量太大.但是可以通过改造切入点` @Pointcut("execution(* " + BuildConfig.PACKAGE_NAME + "..*.*(..))  BuildConfig.PACKAGE_NAME 为项目包名` 使其切入包内所有方法,这样便可以打印所有方法,但是这样又不能满足作为第三方库的问题,因为注解值必须是静态常量,不能动态修改.本项目借鉴了 hugo 库部分思想.在[Github](https://github.com/Thewhitelight/Analysis) 中有使用 AspectJ 实现部分需求,并且可以切入 Kotlin.
Aspectj 本身语法很简单,应该是最容易上手的 AOP 框架,但是对 Kotlin 支持有问题,关于这个可以参考沪江网校对 AspectJ 改造的开源项目,其支持了更多功能,文后参考中有给出相关链接.
##### Javassist
Javassist 是一个开源的分析、编辑和创建 Java 字节码 的类库 .性能较 ASM 差,跟 cglib 差不多,但是使用相对简单.
为了方便引用所以采用 gradle plugin 方式,方便调用者快速添加到原有项目中去.关于编写插件的教程网络有很多,不做过多介绍.

###### TraceExtension 自定义 plugin 配置

``` groovy
class TraceExtension {
    //需要排除的类
    List<String> classExcludes = new ArrayList<>()
    //需要排除的包
    List<String> packageExcludes = new ArrayList<>()
    //是否启用
    def enabled = true
    //项目包名,并非applicationId
    def packageName
    //打印日志的级别
    def logLevel = "d"
}
```

TraceExtension 定义自定义配置类,方便排除一些简单的工具类调用.

###### Trace 

``` groovy
class Trace implements Plugin<Project> {

    @Override
    void apply(Project project) {

        def hasApp = project.plugins.withType(AppPlugin)
        def hasLib = project.plugins.withType(LibraryPlugin)

        if (!hasApp && !hasLib) {
            throw new IllegalStateException("'android' or 'android-library' plugin required.")
        }

        final def variants
        if (hasApp) {
            variants = project.android.applicationVariants
        } else {
            variants = project.android.libraryVariants
        }

        final def log = project.logger
        variants.all { variant ->
            if (!variant.buildType.isDebuggable()) {
                log.debug("Skipping non-debuggable build type '${variant.buildType.name}'.")
                return
            } else if (!project.Trace.enabled) {
                log.debug("Trace is not disabled.")
                return
            }
        }

        project.extensions.create('Trace', TraceExtension)

        println("=== Trace Plugin ===")
        def android = project.extensions.findByType(AppExtension)
        //注册 Transform,注入所要插入的代码
        android.registerTransform(new TraceTransform(project))
    }

}
```

Trace 为自定义的插件,在 release 环境下不需要插入 Log 以免影响性能.

###### TraceTransform 

``` groovy
class TraceTransform extends Transform {

    Project project

    TraceTransform(Project project) {
        this.project = project
    }

    @Override
    String getName() {
        return "TraceTransform"
    }

    /**
     * 需要处理的数据类型
     */
    @Override
    Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS
    }

    /**
     * 需要处理的范围
     */
    @Override
    Set<? super QualifiedContent.Scope> getScopes() {
        // 宿主项目、其子项目及外部引用库
        return TransformManager.SCOPE_FULL_PROJECT
    }

    @Override
    boolean isIncremental() {
        return false
    }

    @Override
    void transform(TransformInvocation transformInvocation) throws TransformException, InterruptedException, IOException {
        super.transform(transformInvocation)
        if (!project.Trace.enabled) {
            return
        }

        def outputProvider = transformInvocation.outputProvider

        transformInvocation.inputs.each { TransformInput input ->
            //宿主项目
            input.directoryInputs.each { DirectoryInput directoryInput ->
                //注入代码
                TraceInject.injectDirCode(directoryInput.file.absolutePath, project)
                def dest = outputProvider.getContentLocation(
                        directoryInput.name,
                        directoryInput.contentTypes,
                        directoryInput.scopes,
                        Format.DIRECTORY)
                FileUtils.copyDirectory(directoryInput.file, dest)
            }

            //第三方jar 虽然对jar没有操作,但是也要输出到out路径 不然运行时会抛错.
            input.jarInputs.each { JarInput jarInput ->
                def jarName = jarInput.name
                def md5Name = DigestUtils.md5Hex(jarInput.file.getAbsolutePath())
                if (jarName.endsWith(".jar")) {
                    jarName = jarName.substring(0, jarName.length() - 4)
                }
                def dest = outputProvider.getContentLocation(
                        jarName + md5Name,
                        jarInput.contentTypes,
                        jarInput.scopes,
                        Format.JAR)
                FileUtils.copyFile(jarInput.file, dest)
            }
        }

    }
}
```

我们为了在方法中添加 Log 就需要赶在 class 文件被转化为 dex 文件之前去修改类,Tranfrom 注册到插件中便会自动添加到 Task 执行序列中，并且正好是项目被打包成dex之前,不需要像 AspectJ 那用手动添加到最后一个 Task 中去.

###### TraceInject 插入 Log

``` groovy
class TraceInject {

    static String TAG = "Trace"
    static ClassPool POOL = ClassPool.getDefault()

    static void injectDirCode(String path, Project project) {
        POOL.appendClassPath(path)
        //添加 Android SDK 路径
        POOL.appendClassPath(project.android.bootClasspath[0].toString())
        File dir = new File(path)
        if (dir.isDirectory()) {
            dir.eachFileRecurse { File file ->
                String filePath = file.absolutePath
                //排除不需要的系统生成class
                if (filePath.endsWith(".class")
                        && !filePath.contains('R$')
                        && !filePath.contains('R.class')
                        && !filePath.contains("BuildConfig.class")) {
                    modifyClass(project, path, filePath)
                }
            }
        }
    }
    static void modifyClass(Project project, String path, String filePath) {
        String classPath
        def packageName = project.Trace.packageName
        filePath = filePath.replace("/", ".")

        if (classExcludes(project, filePath)) {
            return
        }
        if (packageExcludes(project, filePath)) {
            return
        }

        if (filePath.contains(packageName)) {
            int index = filePath.indexOf(packageName)
            classPath = filePath.substring(index, filePath.length())
        } else {
            println("project can not inject")
            return
        }
        //将路径转化为类名
        String className = classPath.substring(0, classPath.length() - 6)
                .replace('/', '.')
                .replace('/', '.')
        CtClass c = POOL.getCtClass(className)
        //类需要解冻后便可以修改,否则无法修改
        if (c.isFrozen()) {
            c.defrost()
        }
        CtMethod[] methods = c.getDeclaredMethods()
        int i = 0
        for (CtMethod method : methods) {
            //排除空方法和 native 方法
            if (method.isEmpty() || Modifier.isNative(method.getModifiers())) {
                return
            }
            i++
            insertTime(project.Trace.logLevel, className.replace(packageName + ".", ""), method)
        }
        c.writeFile(path)
        c.detach()
    }

    static void insertTime(String logLevel, String className, CtMethod method) {
        try {
            int pos = 1

            CodeAttribute codeAttribute = method.getMethodInfo().getCodeAttribute()
            LocalVariableAttribute attribute = (LocalVariableAttribute) codeAttribute.getAttribute(LocalVariableAttribute.tag)
            int size = method.getParameterTypes().length
            String[] paramTypes = new String[size]
            if (attribute == null) {
                return
            }
            for (int i = 0; i < size; i++) {
                //获取入参类型
                paramTypes[i] = method.getParameterTypes()[i].name
            }

            def stringType = POOL.getCtClass("java.lang.String")
            def objType = POOL.getCtClass("java.lang.Object")
            //定义局部变量
            method.addLocalVariable("startTime", CtClass.longType)
            method.addLocalVariable("endTime", CtClass.longType)
            //打印类名
            method.addLocalVariable("className", stringType)
            //打印方法名
            method.addLocalVariable("methodName", stringType)
            //打印行号
            method.addLocalVariable("lineNumber", CtClass.intType)
            //打印返回值
            method.addLocalVariable("returnObj", objType)

            StringBuilder startInjectSB = new StringBuilder()
            
            //*** 省略插入代码

            //方法前插入 Log
            method.insertBefore(startInjectSB.toString())

            StringBuilder endInjectSB = new StringBuilder()
            //*** 省略插入代码

            //方法 return 前插入 Log
            method.insertAfter(endInjectSB.toString())

        } catch (Exception e) {
            e.printStackTrace()
        }
    }
}
```
Javassit 语法相对简单,在看过参考文档和一些简单的资料就可以直接上手.

#### 用法及项目地址

``` gradle
项目 gradle
classpath 'cn.libery.analysis:trace:1.0.1' 
module gradle
apply plugin: 'analysis-trace' 
Trace {
        // 改变配置值后 需要先 clean 项目
        //是否启用插件
        enabled true
        //包名(非 applicationId)
        packageName "cn.libery.analysis.sample"
        //日志等级 和 Logcat 对应
        logLevel "i"
        //排除的类,可以是多个
        classExcludes = ["cn.libery.analysis.sample.Logger",
                        "cn.libery.analysis.sample.Logger2"]
        //排除的包,可以是多个
        packageExcludes = ["cn.libery.analysis.sample.test"]
    }
```

可以参考 Github 中 sample 配置
[Github](https://github.com/Thewhitelight/Analysis)
 [ ![Download](https://api.bintray.com/packages/light/Android/trace/images/download.svg) ](https://bintray.com/light/Android/trace/_latestVersion)

![效果图](https://def-201655.cos.ap-shanghai.myqcloud.com/ic_analysis_trace.png)
-> 打印入参
<- 打印耗时及返回值
#### 参考
[沪江网校 AspectJ](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx)
[Javassist 使用指南](https://www.jianshu.com/p/43424242846b)
[AspectJ 和 Javassist 对比](https://www.jianshu.com/p/44d39585fc20)
[AspectJ 语法](https://www.mekau.com/4880.html)
[hugo](https://github.com/JakeWharton/hugo)
[AspectJ 适配 Kotlin](https://stackoverflow.com/questions/44364633/aspectj-doesnt-work-with-kotlin)