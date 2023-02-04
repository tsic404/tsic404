---
title: "OBS使用指南"
date: 2023-02-01T20:25:19+08:00
slug: "OBS user manual"
tags: [ "open build service", "deepin", "ci/cd" ]
categories: [ '打包', 'ci/cd' ]
draft: false
---

## OBS 介绍
[open build service](https://openbuildservice.org/)简称OBS，是[openSUSE主导开发](https://github.com/openSUSE/open-build-service)的通用构建系统，用于从源码的自动构建和包分发。

假设读者已经了解debian系打包📦相关。本文中对osc的介绍主要作为OBS的CLI接口，本地构建方面可参考[osc 本地构建](#osc-本地构建)。

## OBS 打包流程
本文以deepin(debian系)构建，[deepin OBS实例](build.deepin.com)为例
1. 创建Project
   每个普通账号只有自己的home project。比如：home:tsic404

   也可以可以在Subprojects一栏，创建subproject。比如创建 home:tsic404:ddeOnDebian。
   ![](https://s2.loli.net/2023/02/06/4mSBpOAgrW2fRaD.jpg)
   ![](https://s2.loli.net/2023/02/06/ymsQDnxCF34vuAh.jpg)
   然后
   ```shell
   osc co home:tsic404:ddeOnDebian
   ```
   `home:tsic`项目内容不会包含`home:tsic:ddeOnDebian`,两个是各自 “相对独立” 的project。

2. 添加构建仓库
   在OBS web界面点击Repository，然后add一个Repository到该Project中。
   ![](https://s2.loli.net/2023/02/06/VmgQ59vcUJbodGn.jpg)
   ![](https://s2.loli.net/2023/02/06/gaest56XP3kSnic.jpg)
   如上图add debian sid repo。
   
   或者使用`osc meta prj`修改project使用的仓库。
   ```xml
   <project name="home:tsic404:ddeOnDebian">
      <title/>
      <description/>
      <person userid="tsic" role="maintainer"/>
      <repository name="Debian_Sid">
         <path project="Debian:Sid" repository="standard"/>
         <path project="Debian:ddeExtra" repository="Debian_Sid"/>
         <arch>x86_64</arch>
      </repository>
   </project>
   ```

3. 创建Package
   可以在web点击create package或者使用osc
   ```shell
   cd home:tsic404:ddeOnDebian
   osc mkpac dtkcore
   ```

4. 上传打包文件
   点击dtkcore，然后点击add local file。选择dtkcore相关打包文件。即可上传到OBS的Project中。
   或者使用 osc
   ![](https://s2.loli.net/2023/02/06/zJcxQnu98PdMUpG.jpg)
   ```shell
   cd dtkcore
   cp xxxx/dtkcore/* ./
   osc add *
   osc ci -m "init"
   ```

5. OBS开始构建
   在dtkcore右侧即可看到对应的状态
   点击对应仓库的架构即可看到构建日志，或者使用osc 也可以查看构建日志。
   ```shell
   osc buildlog Debian_Sid x86_64
   ```

**总结**：

   使用OBS进行打包，首先先创建Project，添加构建使用的仓库，然后上传需要打包的package到project中，等待OBS构建完成。Project + 构建仓库决定OBS publish的一个仓库。

## OBS 打包优化(service)及SCM集成
传统方式需要用户自己手动上传打包需要的全部文件xxxx.org.tar.gz xxx.debain.tar.gz xxxx.dsc三件套，这个过程比较麻烦。我们可以使用OBS的service来生成这些必要的文件。

### OBS service介绍
service 简单来说就是一些辅助型的脚本，来协助生成和修改打包所需要的文件。

比如:

使用obs_gbp为git仓库生成debian构建需要的dsc等。

使用set_version动态修改版本号。

使用download_url从指定的url(比如release)下载压缩包等。

service有自己的mode(运行的时机)
service的runmode有
|Mode|是否在OBS server上运行|本地构建时是否运行|service新增文件处理方式|
|-|-|-|-|
|Default|在commit之后|本地构建之前|生成_service:前缀的文件|
|trylocal|Yes|Yes|生成文件会直接merge到commit里面|
|localonly|No|Yes|生成文件会直接merge到commit里面｜
|serveronly|Yes|No|以_service:前缀开始，但是用户本地不会生成|
|buildtime|构建调用dpkg-buildpackage之类的打包工具之前|每次构建之前|
|manual|No|只有OSC命令调用||
|disabled|No|只有OSC命令调用||

注：
1. 在对应时间运行service时，需要此时安装该service。比如trylocal需要本机，server需要server安装，buildtime需要worker安装。比如选择buildtime，但是worker没有对应service的依赖时，构建就会处于unresolvable状态。

2. build.deepin.com提供了[download_files](https://github.com/openSUSE/obs-service-download_files),[download_url](https://github.com/openSUSE/obs-service-download_url),[obs_gbp](https://github.com/openSUSE/obs-service-tar_scm)，[obs_scm](https://github.com/openSUSE/obs-service-tar_scm)，[tar_scm](https://github.com/openSUSE/obs-service-tar_scm)，[set_version](https://github.com/openSUSE/obs-service-set_version)，[tar](https://github.com/openSUSE/obs-service-tar_scm)几种服务。

### 使用service过程
   ```shell
   cd home:tsic404:ddeOnDebain
   vim _service
   ```
   加入以下内容
   ```xml
   <services>
      <service name="obs_gbp">
         <param name="url">https://github.com/linuxdeepin/dtkcore.git</param>
         <param name="scm">git</param>
         <param name="exclude">.git</param>
         <param name="exclude">.github</param>
         <param name="versionformat">@CHANGELOG@@DEEPIN_OFFSET@</param>
         <param name="gbp-dch-release-update">enable</param>
      </service>
   </services>
   ```
   ```shell
   osc add _service
   osc commit -m "init"
   ```
   然后web看到service is running的提示，等待service运行完毕，即可看到对应文件的生成。
   
   使用service可以极大方便使用源代码服务器(git等)的项目打包构建。

### workflow介绍
注：OBS-api需要在2.11版本之后才有workflow功能

workflow 的步骤有
1. branch_package

   ```yaml
   test_build:
      steps:
         - branch_package:
            source_project: deepin:Develop:main
            source_package: %{SCM_REPOSITORY_NAME} 
            target_project: deepin:CI
      filters: 
         event: pull_request
   ```

2. link_package

   ```yaml
   test_build:
      steps:
         - link_package:
            source_project: deepin:Develop:main
            source_package: %{SCM_REPOSITORY_NAME} 
            target_project: deepin:CI

         - configure_repositories:
            project: deepin:CI
            repositories:
               - name: deepin_develop
                 paths:
                  - target_project: deepin:CI
                    target_repository: deepin_develop
                 architectures:
                  - x86_64
                  - aarch64

      filters:
         event: pull_request
   ```

3. configure_repositories

   configure_repositories 如上面所示，用于配置link_package后的project构建仓库。

4. set_flag

   支持的flag有
   - lock (default status disable)
   - build (default status enable)
   - publish (default status enable)
   - debuginfo (default status disable)
   - useforbuild (default status enable)
   - binarydownload (default status enable)
   - sourceaccess (default status enable)
   - access (default status enable)

5. trigger_service

   ```yaml
   commit_build:
      steps:
         - trigger_services:
            project: deepin:Develop:community
            package: %{SCM_REPOSITORY_NAME}
   ```
   重新trigger 该package的service，用于更新commit仓库的代码。

6. rebuild_package

   触发pacakge的rebuild。

### 使用service与workflow

1. SCM + service 持续集成
   osc token 创建SCM token
   github 填写对应的webhook
   推送.obs/workflows.yml到仓库中，该文件定义了OBS workflow的步骤。

## OBS 集成姿势
在package 左侧点击`Submit Package`。
然后填写如下内容。
![](https://s2.loli.net/2023/02/08/bu2pqBN9S1ovnJc.png)
submit 提交。将生成一个request，等待管理员审核。

project管理员在[tasks](https://build.deepin.com/my/tasks)看到所有的请求

点击对应请求，approve或者reject request。

approve后 package将提交到target project中。

## osc 本地构建
deepin 20环境
```shell
sudo apt install osc obs-build rpm debugedit=4.14.2.1+dfsg1.1-1+dde
```
把obs-build设置免密。
```
sudo visudo
```
加入以下内容
```
LOGIN    ALL = NOPASSWD: /usr/bin/build
LOGIN    ALL = NOPASSWD: /usr/bin/osc
```
```
osc co deepin:Develop:main apt
cd deepin\:Develop\:main/apt
osc build xxxxx.dsc
```
即可使用OBS上的仓库进行构建。

### deepin OBS SCM一些常见问题

1. GitHub webhooks未触发

   大概率是网络问题导致，需要在 GitHub webhook 配置中`redeliver`该次失败的请求。

2. GitHub webhooks传递过来但是没有对应仓库和架构的情况返回

   需要到OBS看看是不是构建前的准备工作出现问题，比如service失败，依赖处于 unresolvable，broken。只有成功构建的 succeeded 和失败的 failed 才会返回到GitHub。

## 备注
参考文档

1. OBS 用户手册
   1. https://openbuildservice.org/help/manuals/obs-user-guide/
   2. https://openbuildservice.org/files/manuals/obs-user-guide.pdf
   3. https://openbuildservice.org/files/manuals/obs-user-guide.epub

2. OBS 管理员手册
   1. https://openbuildservice.org/help/manuals/obs-admin-guide/
   2. https://openbuildservice.org/files/manuals/obs-admin-guide.pdf
   3. https://openbuildservice.org/files/manuals/obs-admin-guide.epub

3. suse wiki
   1. https://zh.opensuse.org/Category:构建服务
   2. https://en.opensuse.org/Category:Build_Service

4. osc https://en.opensuse.org/openSUSE:Build_Service_Tutorial

5. OBS 调度 https://wiki.tizen.org/OBS_scheduler_internals

6. OBS 开发 wiki https://github.com/openSUSE/open-build-service/wiki
