---
layout: article
title:  "Bazel PodToBUILD支持git"
tags: "Bazel"
excerpt_type: html
article_header:
  type: overlay
  background_image:
    src: /pics/pic1.JPG
---
介绍了怎么改造Pinterest的PodToBUILD来支持git仓库拉取~~~
<!--more-->
# 背景
从Cocoapod迁移到Bazel可以使用Pinterest开源的工具PodToBUILD进行转换,为每个引用的pod库单独生成对应的BUILD文件,但是在声明中只能支持引用压缩文件不能直接使用git,这里对其做一个改造让其支持对于git源的引用

# 原有组件使用
可以参考上面链接的README,总之是两步

## WORKSPACE添加引用
需要在WORKSPACE文件里面添加对于这个工具的依赖:
``` Python
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "rules_pods",
    urls = ["https://github.com/pinterest/PodToBUILD/releases/download/0.25.2-8a5efa0/PodToBUILD.zip"],
)
```

通过http_archive依赖线上的工具

## Pods.WORKSPACE编写
通过引用rules_pods(PodToBUILD工具的包名),我们可以在根目录声明一个Pods.WORKSPACE文件,里面声明所有的Pod引用(包括引用的Pod依赖的Pod,即申明打平的所有依赖):
``` Python
new_pod_repository(
  name = "PINOperation",
  url = "https://github.com/pinterest/PINOperation/archive/1.0.3.zip",
)
```

这样我们基本上就完成了这个工具的前置条件.当然这里的问题上面也说明了,这里的url不支持使用git仓库地址.
在查阅文档之后,发现new_pod_reposity支持引用本地的仓库:
``` Python
new_pod_repository(
  name = "PINOperation",
  url = "/Path/To/PINOperation",
)
```

那么这里我们可以自定义下载过程,然后调用这个本地的仓库引用即可~

# 改造支持Git
## 分支开发
首先我们需要fork出自己的仓库,然后开发,这里分为三步

### make
rules_pods提供了makefile来编译仓库的可执行文件:
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/03/02/16146919316765.jpg?x-oss-process=image/auto-orient,1/quality,q_90)


当我们执行make之后即可以生成可执行文件用于外部引用

### 二进制上传
需要注意的是这里最终被引用的产物是/bin/RepoTools,但是在你fork仓库之后并不存在这个文件.这是因为在使用的时候拉取下来之后会执行每个里面对应的可执行BUILD文件进行即时的编译.这里为了开发方便在.gitignore里面取消了对于编译产物/bin/RepoTools的忽略,每次修改都会即时编译一个RepoTools进行上传:
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/03/02/16146919907019.jpg?x-oss-process=image/auto-orient,1/quality,q_90)


### 修改工程依赖
当我们开发完分支后,需要一个工程来验证我们的修改.在前面我们会在WORKSPACE文件里面声明对于rules_pods的依赖:
``` Python
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

http_archive(
    name = "rules_pods",
    urls = ["https://github.com/pinterest/PodToBUILD/releases/download/0.25.2-8a5efa0/PodToBUILD.zip"],
)
```

而在同个文件里面可以看到官方提供的对于git仓库的依赖:
``` Python
git_repository(
    name = "build_bazel_rules_apple",
    remote = "https://github.com/bazelbuild/rules_apple.git",
    tag = "0.19.0",
)
```

所以这里为了让工具指向我们开发的仓库,修改依赖为:
``` Python
git_repository(
    name = "rules_pods",
    remote = "xxxxxx", //敏感信息删除了
    branch = "add-git-support",
)
```

这样就完成了对于我们开发分支的依赖

## Git支持开发
这里的思路是,git可以下载仓库,被下载的本地仓库可以被本地依赖,因此这里在下载完之后调用本地依赖的函数指向下载的仓库即可.这里表面的逻辑都被放在/bin/update_pods.py文件里面.
这里下载采用的字段是url,git的地址也可以认为是一个url,因此可以复用该字段.另外参考官方对于git_repository的设计,一般我们引用的都是一个已经被打上版本号的仓库,比如2.0.1之类的,会被作为一个tag来标记特定的commit,这里新增一个tag来进行判断.

### 判断调用curl还是git
在下载的入口处判断应该调用原有的下载逻辑还是走我们的git逻辑:
``` Python
if _is_http_url(url):
    if _is_git_url(url):
        #url是git地址,调用git逻辑
        print("---use git---\n"+url)
        tag = repository_ctx.tag
        _git_clone_remote_repo(repository_ctx, repo_tool_bin_path, target_name, url, tag)
    else:
        #url是http地址,调用原有下载逻辑
        print("---use url---\n"+url)
        _fetch_remote_repo(repository_ctx, repo_tool_bin_path, target_name, url)
#下面都是引用本地仓库逻辑 
elif url.startswith("/"):
    _link_local_repo(repository_ctx, target_name, url)
elif not url.startswith("Vendor"):
    _link_local_repo(repository_ctx, target_name, SRC_ROOT + "/" + url)
```

### Git的调用
``` Python
def _git_clone_remote_repo(repository_ctx, repo_tool_bin, target_name, url, tag):
    dir_path = os.path.dirname(os.path.realpath(__file__))
    cachePath = dir_path + '/BDBazelPodsCache'
    if not os.path.isdir(cachePath):
        os.mkdir(cachePath)
    targetPath = cachePath + '/' + target_name
    podspecPath = targetPath + '/' + target_name + '.podspec'
    cloneCmd = 'git clone'+ ' -q ' + url + ' -b ' + tag + ' ' + targetPath
    if not os.path.isdir(targetPath):
        print("---exec clone cmd---\n"+cloneCmd)
        os.system(cloneCmd)
    else:
        if not os.path.isfile(podspecPath):
            os.removedirs(targetPath)
            os.mkdir(targetPath)
            print("---exec clone cmd---\n"+cloneCmd)
            os.system(cloneCmd)
    
    _link_local_repo(repository_ctx, target_name, targetPath)
```

这里逻辑比较简单,都是对于shell命令的一些调用.
在下载完毕之后,最后一行会调用链接本地库的函数.整个流程就完成了.
需要注意的是这里有一个坑,rules_pods的默认下载目录是/Vendor/target_name,比如你下载Aspects库,那么会被下载到WORKSPACE文件所在的根目录的/Vendor/Aspects目录.但是在你链接本地库的情况下一定不能下载到该目录,因为rules_pods会在这个目录里面创建替身指向原有的文件,在名称相同的情况下,原有的文件会被删除,整个过程会直接报错.

## 结果
在上面的改造后,这里我们就可以用如下的方式来引用一个git仓库了:
``` Python
new_pod_repository(
    name = "AFgzipRequestSerializer",
    url = "xxxxx",  //敏感信息删除了
    tag = "platform0.0.6",
)
```

然后在命令行执行命令:
![](http://mwebpic.oss-cn-shenzhen.aliyuncs.com/2021/03/02/16146920839859.jpg?x-oss-process=image/auto-orient,1/quality,q_90)
成功下载了git仓库的依赖~

# TODO
在改造完成之后发现还是有部分问题,这里记录一下
1. 支持更多的git特性,比如commit branch等(比较好做)
2. 从pod install里面获取的pod包下载URL中,部分下载下来是空文件夹只包含.git,这里可能需要再做修改,会影响自动化的效率(还好是少数)
