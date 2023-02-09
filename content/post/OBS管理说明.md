---
title: "OBS管理说明"
date: 2023-02-06T17:56:38+08:00
slug: "OBS admin manual"
tags: [ "open build service", "deepin", "ci/cd" ]
categories: [ 'ci/cd' ]
draft: true
---

## 结构说明

OBS 实际搭建如图所示

![](https://s2.loli.net/2023/02/07/4Yhe6R7Kcl2BdyC.png)


1. build.deepin.com 云服务器 运行OBS source server, OBS-api

2. back-deepin-main 武汉地区的backend

3. worker 为武汉构建节点,隔绝网络只能访问backend

## 服务说明

### source-server

1. wireguard
    vpn 用于OBS虚拟组网需要
    
    wg0: xxx.xx.x.1/24

    listenport: 51820

2. apache2
    `obs-api`所需web服务器，`obs-api`为`ruby`语言编写。
    
    对应位置在`/srv/www/obs/api/`,当更新OBS后需注意[README.UPDATERS](https://github.com/openSUSE/open-build-service/blob/master/dist/README.UPDATERS)中所需要的操作。(数据模型更新等

3. obs-delayedjob-queue-xxx
    
    OBS 异步延时job系列，比如scm反馈，web monitor更新等。

4. obsdeltastore
    
    OBS 增量存储

5. obsredis

    OBS redis缓存，需配合redis服务器使用。

6. obsservice

    OBS 服务。

7. obsservicedispatch

    OBS service 转发，转达到bakcend。obsispatch用于将job转发到worker。名称相似功能不同。

8. obssrcserver

    OBS 源码服务，提供5352端口调用，用于获取source server上的源码，proejct配置等。

分区配置

BSConfig.pm
```perl

our $partitioning = [ 'deepin:CI' => 'back-deepin-main',
                       'deepin:'    => 'back-deepin-community',
    ];

our $partitionservers = {
    'back-deepin-main' => 'http://back-deepin-main-ip:5252',
    'back-deepin-community' => 'http://back-deepin-community-ip:5252',
    };
```


### back-deepin-main

1. wireguard

    wg0: xxx.xx.x.2/24

2. nginx

    用于反代obssrcserver和将back-deepin-main上的publish 提供访问。

3. obsdispatcher
    用于将job等转发到obsworker

4. obsnotifyforward
    
    将obsworker的事件进行转发。

5. obspublisher

    OBS进行publish需要的服务。

6. obsrepserver

    OBS仓库服务，提供5352调用，用于worker获取worker脚本，获取打包需要的deb，返回构建结果和状态等。

7. obsscheduler

    OBS调度器，其中支持的每个架构都会启动一个调度器。每个调度器根据当前仓库情况计算需要构建的pacakge。

8. obswarden

    OBS worker的监管服务。

9. teamd@team0

    将四个千兆网口聚合解决backend的网络压力
    
    配置来源 https://documentation.suse.com/smart/linux/html/task-configure-network-teaming/index.html

### workers

1. obsworker
    
    需要配置`/etc/[sysocnifg|default]/obsworker`中，一些常见配置说明。
    |配置项|说明|
    |-|-|
    |OBS_SRC_SERVER|OBS源码服务器，用于获取source server上面的资源(打包需要的源码，项目的配置信息等)。
    |OBS_REPO_SERVERS|OBS的仓库服务器，用于从该主机上获取打包需要的依赖，并把构建结果等上传到该服务器上。
    |OBS_WORKER_INSTANCES|该worker并发构建的最大数量。
    |OBS_WORKER_DIRECTORY|该worker构建时，buildroot所在路径。
    |OBS_WORKER_JOBS|该worker单个构建时使用的最大线程数，会以--max-parallel参数形式传递过去。
    |OBS_RUN_DIR|OBS worker的工作路径，包含从repserver上拉取的worker相关代码(boot),对应构建(上面INSTANCES)的记录等。
    |OBS_CACHE_DIR|worke的缓存，可解决backend网络流量过大问题。
    |OBS_CACHE_SIZE|worker缓存大小。
    |OBS_ARCH_TYPE| worker的架构。
    等。

## 加入指南

1. 加入obs虚拟局域网

    wireguard 组网，本地生成key，pubkey需要加到云服务器上。

    ```shell
    cd /etc/wireguard/
    umask 077 | wg genkey | tee privkey | wg pubkey > pubkey
    vim wg0.conf
    ```
    加入类似一下内容。
    ```
    [Interface]
    PrivateKey = 本机生成的privkey
    Address = xxx.xx.x.x/24

    [Peer]
    PublicKey = 云服务器的pubkey
    Endpoint = 云服务器ip:云服务器端口
    AllowedIPs = xxx.xx.x.x/24
    PersistentKeepalive = 25
    ```
    ```shell
    sudo systemctl enable --now wg-qucik@wg0
    ```

2. 修改backend并配置xxx项目到该backend

    修改`/usr/lib/obs/server/BSConfig.pm`

    ```perl
    # obs-api ip地址
    my $frontend = xxx.xx.x.x;

    our $ipaccess = {
    '^::1$' => 'rw',    # only the localhost can write to the backend
    '^127\..*' => 'rw', # only the localhost can write to the backend
    "^$ip\$" => 'rw',   # Permit IP of FQDN
    # backend 虚拟组网网段,backedn需要有rw权限
    "^xxx.xx.x.*" => 'rw',   # Permit IP of FQDN
    '.*' => 'worker',   # build results can be delivered from any client in the network
    };
    # source server的地址
    our $srcserver = "http://source-server-ip:5352";
    our $reposerver = "http://backend-ip:5252";
    our $serviceserver = "http://source-server-ip:5152";

    #worker 链接时，使用的链接地址
    # 由于采用了nginx反代source server,所有两者均指向backend-ip
    our $workersrcserver = "http://backend-ip:5352";
    our $workerreposerver = "http://backend-ip:5252";

    # redis 服务器地址
    our $redisserver = "redis://source-server-ip:6379";

    #需要将构建结果放到https://ci.deepin.com/repo/obs/时，需要将publish仓库捅不到deepin的backend上
    #our $stageserver = 'rsync://back-deepin-main-ip/repo/';

    #本backend名称
    our $partition = 'back-deepin-main';

    #一下配置需要在source serve上配置，指定那些project 会到那些backend上。
    #规划分区的方式，让不同的backend承担不同的project。
    # our $partitioning = [ 'home:' => 'home',
    #                       '.*'    => 'main',
    #                     ];
    #
    # our $partitionservers = { 'home' => 'http://home-backend-server:5252',
    #                           'main' => 'http://main-backend-server:5252',
    #                         };
    ```

3. 启动backend所需服务，并把workers加入到back-end中

启动nginx反代云服务器并设置缓存，用于减轻云服务器压力。
```
proxy_cache_path /cache levels=1:2 keys_zone=obssrc:10m inactive=365d max_size=200g;

server {
    listen       5352;
    location / {
    	   proxy_ignore_headers Cache-Control;
    	   proxy_cache obssrc;
    	   proxy_buffering on;
	   proxy_cache_valid any 30m;
	   proxy_cache_lock on;
           proxy_cache_background_update on;
	   proxy_cache_use_stale error timeout updating http_503;
           proxy_pass http://xxx.xx.x.x:5352;
    }
    add_header X-Cache-Status1 $upstream_cache_status;
}
server {
    listen       5152;
    location / {
           proxy_pass http://xxx.xx.x.x:5152;
    }
}
```

## OBS 一些资源说明

1. source server 资源管理

    souce server所有资源均在`/srv/obs/`目录下。

    ```shell
    xxx@xxx:~> ls /srv/obs/
    build              db         gnupg  log                  projects     run      trees
    certs              diffcache  info   MySQL                remotecache  service  upload
    configuration.xml  events     jobs   obs-default-gpg.asc  repos        sources  workers
    ```
    一些路径说明
    1. configuration.xml OBS-api配置文件，其中配置优先于/etc/sysconfig/obs-server,其中可配置OBS支持的架构等。
    2. projects/

        OBS project的配置，meta等信息保存路径。

    3. sources/

        package上传的打包文件存放路径。

2. back end 资源管理

    back end所有资源在`/srv/obs`目录下
    ```shell
    xxx@xxx:~> ls /srv/obs
    build  configuration.xml  diffcache  events  info  log    projects     repos  service  tmp    upload
    certs  db                 dods       gnupg   jobs  MySQL  remotecache  run    sources  trees  workers
    ```
    一些路径说明
    1. build/

        build下为对应的Project构建中间结果。
        /srv/obs/build/{project name}/{repo name}/{arch}/
        /srv/obs/build/{project name}/{repo name}/{arch}/:full/这个路径为存放用于构建的deb。
        
        当首次构建deepin:Develop:main能希望自我迭代时，须在该路径下存放能够组成base的deb。

        /srv/obs/build/{project name}/{repo name}/{arch}/:meta/
        /srv/obs/build/{project name}/{repo name}/{arch}/:full/*.meta
        /srv/obs/build/{project name}/{repo name}/{arch}/{package}/.meta.success
        
        OBS调度的meta记录。
    
    2. worker/
        
        连接该backend的所有状况，分为buildding,idle,down等几个子目录文件夹。

    3. jobs/
        
        正在进行构建的job

## 一些问题说明

1. bs_admin
    
    OBS backend上提供了`/usr/lib/obs/server/bs_admin`命令行工具，用于管理worker，project等，比如清理被OBS认为是badhost的worker，重新check project等。

2. prjconf

    一个Project的配置存储为prjconf,该prjconf定义了一个package build的_buildenv。

    比如依赖有choice时，使用Prefer决定实际构建使用的Package。

    更多说明请参考

3. configuration.xml

    其中的一些配置会写入到数据库的configurations表中。比如download_url,registration等
