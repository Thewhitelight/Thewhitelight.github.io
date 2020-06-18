title: 提取 Git Messages 生成 ChangeLog
tags:
  - Git
  - Shell
categories:
  - 技术
date: 2019-11-18 18:25:00
---
# 需求
随着版本迭代越来越多,我们需要每次手动修改 ChangeLog,多人维护项目则容易忘记填写,并且组件化后每个模块多人维护,容易忘记填写.Module 里填写了 ChangeLog 但是 App 壳项目也要填写相同内容,也是比较麻烦的事.所以需要引入自动化来降低我们手动维护的成本.
<!-- more -->
# 解决方案
主要问题是解决手动维护的问题,所以解决办法自然而然就是使用自动化方案,那么怎么交给自动化处理呢,自然是交给 Git 和 Shell 了.我们通过获取最近两个 git tag 之间所有的 commit message,然后筛选符合规则的 messge,把这些 message 合成然后生成 ChangeLog,这样就生成 Modlue 的 ChangeLog,然后我们可以把 ChangeLog 文件推到一个指定仓库里,然后通过特定脚本合成 App 的 ChangeLog.为了规范 commit message ,所以也需要用到 git hook 在 commit 的时候去检查 message 是否符合我们的规范.
## commit message 规范
要生成 ChangeLog 就需要填写符合规则的 message,我们参考[Angular Commit 规范](https://github.com/angular/angular/commits/master),然后做了相应的简化.
commit message 格式:
```
type: subject

body

footer
```
### header
Header 部分只有一行，包括两个字段：type（必需）和 subject（必需）。

#### type 用于说明 commit 的类别，只允许使用下面3个标识。
* feat 新功能（feature）
* fix 修补bug
* other

如果 type 为 feat 和 fix，则该 commit 将肯定出现在 Change log 之中。
**如果 type 为 feat 和 fix，则该 commit 将肯定出现在 Change log 之中。**

#### subject 是 commit 目的的简短描述，不超过50个字符。
以动词开头，使用第一人称现在时，比如 change，而不是 changed 或 changes
第一个字母大写
结尾不加句号
### body 描述为什么修改, 做了什么样的修改, 以及开发的思路等等

### footer 放 Breaking Changes 或 Closed Issues

## Git Commit 工具
Message规则很简单,如果养成习惯很容易上手,如果刚开始不适应可以使用下面的方法
### 配置 git 模版
1. 在git全局配置里进行设置，linx/mac 进入文件`.gitconfig`
```
$ vi ~/.gitconfig
```
2. 若不存在[commit] template，则设置如下
```
[commit]
        template = /Users/<name>/.myCommitMsg
```
3. 设置模板完毕后，下一步进行模板内容的修改
```
$ vi /Users/<name>/.myCommitMsg
```
粘入以下内容保存即可
模版可以根据自己喜好随意配置,但是 Header 的格式和内容一定要保证正确
使用时修改为自己提交的内容
```
type: subject

body

footer

# - type: 
#    feat(新特性), 
#    fix(修改问题), 
#    other(其他任何修改), 
# - subject
#   提交描述
# - body:
#    描述为什么修改, 做了什么样的修改, 以及开发的思路
# - footer:
#    放 Breaking Changes 或 Closed Issues
```
4. 若使用 terminal 则使用`git commit`

若使用 sourcetree 等 git 管理软件，则需要重启软件才能生效。
### Sourcetree
设置->提交模版->自定义,可以使用填入上述模版,并且只针对此项目生效.如果使用全局配置,参考上条配置方式.
### git cz
安装[git cz](https://github.com/commitizen/cz-cli)
``` 
npm install -g commitizen cz-conventional-changelog
echo '{ "path": "cz-conventional-changelog" }' > ~/.czrc
```
使用`git cz`代替`git commit`命令,会出现提交选项,按规则提交就可以,如下所示

![](https://raw.githubusercontent.com/commitizen/cz-cli/master/meta/screenshots/add-commit.png)

然后在根据对应的提示下,写入内容即可完成一次提交
``` shell
➜  sh git:(master) ✗ git cz
cz-cli@4.0.3, cz-conventional-changelog@3.0.1

? Select the type of change that you're committing: feat:        A new feature
? What is the scope of this change (e.g. component or file name): (press enter t
o skip)
? Write a short, imperative tense description of the change (max 94 chars):
 (4) test
? Provide a longer description of the change: (press enter to skip)

? Are there any breaking changes? Yes
? A BREAKING CHANGE commit requires a body. Please enter a longer description of the commit
itself:
 -
? Describe the breaking changes:

? Does this change affect any open issues? No
```
但是此格式不符合我们的规范,已经不能使用.
### Android Studio Commit Message 插件
使用[Commit Message Template](https://plugins.jetbrains.com/plugin/9364-commit-message-template)在`Settings > Tools > Commit Message Template `中填入文章开始的格式.
### git cz
安装[git cz](https://github.com/commitizen/cz-cli)
``` 
npm install -g commitizen cz-conventional-changelog
echo '{ "path": "cz-conventional-changelog" }' > ~/.czrc
```
但是此格式不符合我们的规范,已经不能使用.
### Android Studio Commit Message 插件
使用[Commit Message Template](https://plugins.jetbrains.com/plugin/9364-commit-message-template)在`Settings > Tools > Commit Message Template `中填入文章开始的格式.
### git cz
安装[git cz](https://github.com/commitizen/cz-cli)
``` 
npm install -g commitizen cz-conventional-changelog
echo '{ "path": "cz-conventional-changelog" }' > ~/.czrc
```
使用`git cz`代替`git commit`命令,会出现提交选项,按规则提交就可以,如下所示
但是此格式不符合我们的规范,已经不能使用.
### Android Studio Commit Message 插件
使用[Commit Message Template](https://plugins.jetbrains.com/plugin/9364-commit-message-template)在`Settings > Tools > Commit Message Template `中填入文章开始的格式.
这样就可以在提交时点击选择模版,我们基于模版修改就可以了.

## 生成 Module 的 ChangeLog
简单归纳步骤如下:
1. git tag 获取最近的两个tag
2. 获取两个tag之间的commit message
3. 使用脚本清洗 log,筛选符合规则的 message
4. 生成changelog.md
脚本使用 shell 编写,所以对其需要有稍微的了解.

### 获取最近两个 tag
如果未输入两个 tag,则获取最近的两个 tag
``` shell
# 获取最近提交的两个 tag
function getAutoTag(){
# 根据日期排序获取所有tag
current_git_tags=$(git tag --sort=taggerdate)
for word in ${current_git_tags}
do
    tags[temp]=${word}
    ((++temp))
done
# 获取最近的两个 tag
tag1=${tags[temp-2]}
tag2=${tags[temp-1]}
}
```
###  获取两个 tag 或者 commitId 之间所有的 message
``` shell
# 获取两个 commitId 之间的 message
function getInputCommitId(){
tag2=${input2}
tag1=${input1}
# 获取的 message 区间为[)
git log --pretty=format:"%s" ${tag2}...${tag1} >commits.txt
# 写入空格
echo  >>commits.txt
# 获取第二个输入 commitId 的 message
git log --pretty=format:"%s" ${tag1} -1 >>commits.txt
}

# 获取两个tag之间所有的 message
function getTagMessages(){
getAutoTag
echo $tag2  $tag1
if [[ "" = ${tag1} ]]
    then
# 如果为获取到后提交的 tag,则获取先提交 tag 以前所有的 message,并写入到 commits.txt文件里
    git log --pretty=format:"%s" ${tag2} >commits.txt
else
git log --pretty=format:"%s" ${tag2}...${tag1} >commits.txt
fi
# 将两个tag之间的所有commit message写入commits.txt
}
```
这样我们就获取了所有的 message并写入到了commits.txt文件中,现在需要做的就是清洗提交记录
### 筛选符合规则的 message
上一步我们把所有的 message 写入到了 commits 文件中去,这样我们就可以把文件读出来进行筛选,获得符合规范的 message
``` shell
# 使用正则判断 commits.txt 每行开头是否符合我嗯的规则,并生成feat.txt和fix.txt
function generateDoc(){
    # $1 代表方法 第一个入参数,方法参数为每行文本
    if [[ "$1" =~ ^feat.*:$ ]]
        then
        # msg 代表 feat: 后所有的文本
        msg="${@:2}"
        echo ${msg} >> feat.txt
    elif [[ "$1" =~ ^fix.*:$ ]]
        then
        # msg 代表 fix: 后所有的文本
        msg="${@:2}"
        echo $msg >> fix.txt
    fi
}

# 读取 commits.txt 每行文本
while read line || [[ -n ${line} ]]
do
    message=${line}
    generateDoc ${message}
done < commits.txt
# 删除 commits.txt
$(rm -rf commits.txt)
```
### 合成 ChangeLog
通过分别读取 feat.txt 和 fix.txt 文件合成 ChangeLog
``` shell
# 生成tag标题
# inputType=2 代表使用 commitId 生成 log
if [[ ${inputType} = "2" ]]
    then
    branch=$(git symbolic-ref --short -q HEAD)
    time=$(git log --pretty=format:"%ai" ${tag2} -1)
    echo "## $tag2 - $tag1 $branch ($time)" >>CHANGELOG.md
else
# 使用 tag 生成 log
    time=$(git log -1 --format=%ai ${tag2})
    echo  "## $tag2 ($time)">>CHANGELOG.md
fi

# 生成Features行
echo "### Features" >>CHANGELOG.md
while read line
do
    echo "* "${line} >> CHANGELOG.md
done < feat.txt
$(rm -rf feat.txt)

# 生成Fixs行
echo "### Fixs" >>CHANGELOG.md
while read line
do


# 判断文件是否存在
if [[ -f "./CHANGELOG.md" ]]
then
    echo CHANGELOG.md is exsit
else
    echo '# ChangeLog' >CHANGELOG.md
fi

# 读取 ChangeLog 文件,删除重复输入一直的内容,使用第一个输入的参数作为锚点对比
while read text
do
    ((++i))
    if [[ ${text} =~ "## $tag2" ]]
        then
        start=$i
    elif [[ $tag1 != "" && ${text} =~ "## $tag1" ]]
        then
        end=$i
    fi
done < CHANGELOG.md

# ${start} 为起始行
# ${end} 为结束行

if [[ "" = "$end" ]]
    then
    if [[ "" != "$start" ]]
        then
        # 如果没有结束行,则删除起始行以后所有的内容
        $(sed -i "" "$start,\$d" CHANGELOG.md)
    fi
else
    ((--end))
    # 应换为插入
    if [[ "$start" -gt "$end" ]]
        then
        $(sed -i "" "$start,\$d" CHANGELOG.md)
    else
    # 如果有结束行 则删除之间所有的内容
        $(sed -i "" "$start,${end}d" CHANGELOG.md)
    fi
fi

```

# 集成工具
因为需要提供 git template、检查 git message 是否符合规范和生成 CHANGELOG并且脚本都是在客户端,如果通过手动拷贝的方式分享给组内成员会很麻烦,并且后续更新也是很大的问题.所以通过 curl 和 wget 等命令,直接执行相应脚本,可以极大的方便接入脚本工具.

# 总结
由于自己也是初次接触 shell 可能写的不算好,边学边写完成了这个需求,shell 对于数据的处理还是很方便,可以做很多有意思的事情,提高开发效率.最近项目打算迁移 androidx,由于模块太多使用 AS 自带插件逐个升级,太麻烦所以肯定也需要通过脚本直接修改源码升级,正好也加强学习 shell.
本次只是实现了 Module 的 ChangeLog 生成,但是 App 的还未完成,目前考虑的方案有两种
* 在生成脚本里把文件传到远程指定目录下,然后通过脚本提取所有 module 的文件,再重新生成 App 的 ChangeLog
* 通过脚本获取 App 的所有业务依赖,然后通过依赖再获取 git 地址,通过 git 地址下载到 module 的 ChangeLog,在重新生成 App 的 ChangeLog 
两个方法个有好处吧,由于其他事情耽误目前还没开始做,方案记录下来有空实现.

# 参考
https://ruby-china.org/topics/15737

https://juejin.im/post/5afc5242f265da0b7f44bee4

[git message lint 工具](https://commitlint.js.org/)
