---
title: 使用 api 上传产物至 jcenter
date: 2020-08-31 21:24:29
tags:
    - Android
    - Python
    - Jcenter
    - maven
categories: 技术
---
Jcenter 是目前最大的 maven 源，通过 AS 创建的项目自身就使用 Jcenter，当我们想上传自己的 aar 至 jcenter 时都会选择官方提供的插件，它可以很方便的将我们的产物上传至 jcenter，然后在产物的详情页里点击 **Add to jcenter** 就可以提交审核到公共源里。
# 需求
由于项目开源需要将某些产物从私有 maven 发布到 jcenter，但是如果依赖插件打包就有太多的组件要打包历史版本，显然看起来时不可行的，所需要研究怎样把私有 maven 的产物发布到 jcenter。
<!-- more -->
# 方案
首先就想到通过 api 调用，使用 nexus 的 api 下载产物，然后通过 jcenter 的 api 上传产物。
## 下载脚本
通过[搜索api](https://help.sonatype.com/repomanager3/rest-and-integration-api/search-api)即可搜索相关组件，通过对[返回结果](https://help.sonatype.com/repomanager3/rest-and-integration-api/search-api#SearchAPI-SearchAssets)的分析,我们只需要获取`downloadUrl`字段就可以拿到下载地址，也可以通过对`downloadUrl`的获取下载产物的内容来做是否下载的判断。通过`continuationToken`我们也可以获取分页信息。
根据上面的分析就可以写出一个简单的下载脚本

``` python
def downloadArtifact(items):
    for item in items:
        url = item['downloadUrl']
        # 下载所需类型的产物
        if url.endswith(".jar") or url.endswith(".aar") or url.endswith(".pom"):
            path = item['path'].rsplit('/', 1)[0]
            # 使用 wget 下载产物
            os.system("wget -P "+path+" "+url)
            # 以下代码格式化数据为上传作准备
            path = path.split('/')
            name = ''
            for i in range(0, len(path)-2):
                name += path[i]
                if i != len(path)-3:
                    name += "."
            artifactType = ""
            if url.endswith(".jar"):
                artifactType = "jar"
            elif url.endswith(".aar"):
                artifactType = "aar"
            if artifactType != "":
                print(url)
                upload(name, path[-2], path[-1], artifactType)

r = requests.get('https://***/service/rest/v1/assets?repository=maven-snapshots',
                    auth=('username', 'password'))
resp = json.loads(r.text)
# 解析item下载
downloadArtifact(resp['items'])
# 获取分页信息，不为null则表示有下一页
pagination = resp['continuationToken']
```
通过上面的代码就可以下载到我们所需的产物。

## 上传脚本

上传产物到 jcenter 可以使用 [Bintray REST API](https://www.jfrog.com/confluence/display/BT/Uploading+Using+APIs)，通过阅读文档可知上传需分两步，创建 Package、上传产物、发布产物。

``` python
baseApi = 'https://api.bintray.com/'
# 在 user->profile->API Key 即可获得
apiKey = "***"
# https://bintray.com/ 右上角显示的名称
subject = '***'
# 仓库名
repo = '***'


def upload(groupId, artifactId, version, artifactType):
    groupPath = groupId.replace(".", "/")
    artifactPath = "%s/%s/%s" % (groupPath,
                                 artifactId, version)
    pomPath = "%s/%s/%s" % (groupPath, artifactId, version)    

    # 根据 artifactIdid 创建 package
    # createPackageParams 里面的参数为固定结构
    # vcs_url 包含 源码的 git 链接
    # licenses 为固定协议，可以通过 package 详情查询
    createPackageParams = {
        "name": artifactId,
        "vcs_url": "***",
        "licenses": ["Apache-2.0"]}
    createPackageParams = '\''+json.dumps(createPackageParams)+'\''
    caretePackageComm = 'curl -H "Accept: application/json" -H "Content-type: application/json" -X POST -d %s -u%s:%s %spackages/%s/%s' % (
        createPackageParams, subject, apiKey, baseApi, subject, repo)
    print(caretePackageComm)
    # 调用 curl 
    os.system(caretePackageComm)
    print()

    def uploadArtifact(filePath, fileName):
        uploadComm = 'curl -T %s/%s -u%s:%s %scontent/%s/%s/%s/%s/%s/%s' % (filePath, fileName, subject, apiKey, baseApi, subject,
                                                                            repo, artifactId, version, filePath, fileName)
        print(uploadComm)
        print(fileName)
        os.system(uploadComm)
        print()

    if (artifactType != ""):
        # 上传产物 aar/jar souces.jar pom
        artifactFileName = '%s-%s.%s' % (artifactId,
                                         version, artifactType)
        uploadArtifact(artifactPath, artifactFileName)
        sourceJarFileName = '%s-%s-sources.jar' % (artifactId, version)
        uploadArtifact(artifactPath, sourceJarFileName)
        pomFileName = '%s-%s.pom' % (artifactId, version)
        uploadArtifact(pomPath, pomFileName)
        
        # 发布产物 其他用户可以看到，但还未同步到 jcenter
        publishComm = 'curl -X POST  -u%s:%s %scontent/%s/%s/%s/%s/publish' % (
            subject, apiKey, baseApi, subject, repo, artifactId, version)
        print(publishComm)
        os.system(publishComm)
        print()
```
上传脚本其实很简单，需要仔细阅读 api 文档便可写出，在编写脚本过程中需要注意几点。

**创建 package**
* vcs_url 应该为项目的 git 源码
* licenses 为必填且是选择项，不可随意填写
* 重复创建也没关系，会失败不会影响原有 package

**上传产物**
* 第一次提交 AddToJcenter 是要求必须有 ***-sourcers.jar,所以必须上传该文件，如果已经同步过公开 jcenter 源，可以不上传

**发布产物**
意味着所有用户都可见，但是不在公共jcenter源，如果其他要使用需要添加
``` 
  maven { url "https://dl.bintray.com/{subject}/{repo}" }
```
才能引用到该组件

# 注意
AddToJcenter 需要官方审核，在文档中没有找到相关接口，所以只能通过 Web 操作才可以