---
title: "linux 下使用wayland合成器将插件分进程"
date: 2024-03-15T17:45:17+08:00
tags: [ "Qt", "wayland", "qml", "c++", "deepin"]
categories: ["开发日志"]
draft: false
---

## 背景

1. 为什么要将插件分进程

    原有的dde-dock经常因为插件问题导致假死崩溃卡顿等问题，插件与dock在相同进程就避免不了因为插件代码质量等导致的同生共死，所以有将插件独立进程出去将插件的影响范围降低到最小，只影响自己。

2. 分进程有什么好处与坏处

    分进程后带来的好处可以从以下几个方面体现：

        1. 兼容性：只要确定了插件进程与dock进程通信的方式，可以使用这种通信方式都可以被dock当作插件加载上来。无论widget，qml，qt5/6还是其他语言等。
   
        2. 流畅性：dock自身更加专注于dock本身的业务逻辑，不需要发生变化时去更新插件的状态，也不会被插件阻塞住。
   
        3. 稳定性：dock进程只有dock自身，不再受插件的影响。

    缺点方面：

        最主要的一点就是总体的系统资源占用会比之前稍高。
        开发复杂度和维护复杂度都会上升。

3. 怎么去分进程

    插件的主要作用：

        1. 用于显示，显示插件的UI部分。
        2. 获取输入，获取用户的鼠标或者键盘输入，并作出对应的响应。

   确定一种通信方式可以让 dock 和插件通信，直接使用 socket 方式不能通信UI相关的信息，共享 buffer 来实现显示 = 换个类似的上层封装，可以接受 UI 相关 ---“显示服务器！”

   linux上的显示服务器有xorg，wayland等。采用 wayland 的原因是现代化，方便扩展，并且 Qt 做好了 qt-wayland 的上层封装很方便开发者去开发Wayland相关。

## wayland 简单介绍

1. wayland 是什么

   wayland 是x11的取代品，wayland是一套应用程序用于获取用户输入和显示的协议。[wayland](https://wayland.freedesktop.org/)

   正好与插件的需求相切合，用于显示和获取输入。

   显示和输入的两方面问题

   wayland中如何显示？        surface

   如何暴露插件属性和插件交互？ shell-surface

2. wayland 有什么好处

   1. 更现代化的设计
   2. 更低的延迟
   3. 更好的性能
   4. 更好的安全性
   5. 更简单的代码库

3. shell-surface介绍

    显示比较好理解，surface 是wayland中需要渲染的一片区域。

    那 shell-surface 是什么呢？

    shell-surface 是具有元数据属性的 surface。它可以具有各种属性比如窗管中定义的窗口标题，窗口类型，窗口状态等，还具有布局和排列的能力和交互响应等。

## dock 插件接口介绍

1. dock 提供给插件的接口

    [pluginproxyinterface.h](https://github.com/linuxdeepin/dde-dock/blob/32bd005492009aaa7e9ea2d59237fa8bf9a1f75c/interfaces/pluginproxyinterface.h)

2. 插件提供给dock的接口

    [pluginsiteminterface.h](https://github.com/linuxdeepin/dde-dock/blob/32bd005492009aaa7e9ea2d59237fa8bf9a1f75c/interfaces/pluginsiteminterface.h)

3. 之前的工作流程

    dock 通过 pluginloader 将插件加载到 pluginmanager，然后对插件进行初始化。

    dock 再根据 pluginmanager 加载进来的插件，去加载对应类型区域的部分。

    比如：

    - 工作区预览（Fixed类型插件） 将显示在 dock 定义的 Fixed 区域
    - 电源按键（system类型插件）显示在 dock 定义的 System 区域
    - 回收站（Tools类型插件）显示在 dock 定义的 Tool 区域
    - 托盘（tray类型）显示在 dock 定义的 Tray 区域
    - 快捷面板将被快捷面板加载，获取到各个插件的快捷面板里面的UI。

4. shell-surafce 与 插件的相似之处

    插件需要按照一定的顺序显示，shell-surface 可以根据窗口的属性进行布局和排列。

    插件需要响应用户的输入，shell-surface 具有交互性和事件处理。

    所以接下来需要定一个自定义类型的 shell surface 来实现dock 插件管理的需求。

## 通过 shell surface 实现插件加载和通信流程

1. wayland 协议介绍

    wayland 是一套应用程序用于获取用户输入和显示的协议。[wayland](https://wayland.freedesktop.org/)

    现有的wayland显示协议都不满足dock插件管理的要求，所以定义一套dock的插件管理协议，用于dock插件的显示和响应。

2. 确定对应的shell-surface和对应的wayland协议

    根据上面的内容，输入转换

        插件向dock的接口    ->      wayland request
        dock向插件的接口    ->      wayland event

    得出下面一份简单的 [Wayland 协议](https://github.com/linuxdeepin/dde-shell/blob/7d3437492afc37c2d5a6bd908fb7a7fce501581f/panels/dock/dockplugin/protocol/dock-plugin-manager-v1.xml)，仍有需要补充部分

3. 制定协议后的工作流程

   1. 加载流程

       1. widget 插件由 widget 的loader进行初始化

       2. 实现一个 widgetplugin 将初始化后的插件和协议的具体实现连接起来。

       3. 将 每个widget 初始化称独立窗口，使用窗口的显示和隐藏控制 shell-surface 的创建和隐藏达到控制插件的注册后的显示与否。

   2. 通信流程

       1. 插件通过 wayland 的request 向 dock 发送插件提供的接口

       2. dock 通过 wayland 的 event 向插件发送 dock 提供的能力

        比如点击事件：

            1. 点击 dock 上面的 item， item 通过 wayland 产生一个自定的 event

            2. 该 event 通过 wayland 发送到协议的客户端实现，调用对应的event 处理接口，将事件发送到 widgetplugin实现中。

            3. widgetplugin收到转发后的 点击事件，调用插件对应的点击接口

            4. 插件响应对应的点击行为。

   3. 通过 wayland 后的插件时序图

    ![时序图](https://s2.loli.net/2024/03/18/bVyNdEwWLQBtZTo.png)

## 使用Qt wayland 实现协议

0. 工具 qtwayland-scanner 根据协议生成胶水代码

    1.1 客户端自动填充好 request 相关接口的调用逻辑，需要实现收到对应 event 后的处理和 request 如何暴露对外的方式。

    对于 dock 插件来说， event 一般就是 dock 的状态变更，将其关联到对应的信号处理即可，将协议定义的 request 与旧插件接口相关的调用关联起来即可。

    1.2 服务端自动填充好 event 相关接口的调用逻辑，需要实现收到对应的 reqeust 后的处理逻辑和 event 暴露对外的方式。

1. server端（也就是dock本体的合成器）实现

    server 是作为一个 QWaylandCompositorExtension 实现，需要将PluginManager暴露到QML中用于注册到 wayland 合成器中作为受支持的协议。并且需要关联 dock 对应的属性到上面，用于发送对应的 dock 状态变化后的事件。

    通过 Qt的 QML_ELEMENT 和 Q_COMPOSITOR_DECLARE_QUICK_EXTENSION_CLASS()
    宏注册到 QML 中作为QWaylandCompositorExtension

    PluginSurface 是由 PluginManager 控制创建，不需要 QML 中创建暴露，但是 PluginSurface 需要响应对应的输入行为，所以一些输入行为接口需要设置成 Q_INVOKABLE 由 QML 直接调用。

    细节：

    1. 实现 DockPluginManager
        ```c++
        class DockPluginManager : public QWaylandCompositorExtensionTemplate<DockPluginManager>, public QtWaylandServer::dock_plugin_manager_v1
        ```

    2. 实现胶水代码中的虚函数，暴露属性并关联到 event 上，用于向插件通知 dock 的状态变更。

    3. 实现 surface 创建流程，根据 QWaylandSurface 和 QWaylandResource 创建 PluginSurface。

    4. 实现 PluginSurface
        ```c++
        class PluginSurface : public QWaylandShellSurfaceTemplate<PluginSurface>, public QtWaylandServer::dock_plugin_surface
        ```
    5. 定义对应的属性和对外暴露，用于接收插件上报的属性。
    6. 定义对应的信号，用于插件发生对应的 request 后转发给 dock。
    7. 实现对外的接口，将输入事件转成对应的 event 发送给插件。

2. client端（也就是插件）实现

    由于 widget 插件是使用 Qt5 编写，所以注册方式采用 Qt5 的方式。

    将 PluginManager 通过 QWaylandShellIntegration 和 QWaylandShellIntegrationPlugin 注册到 wayland 的createSurface上。

    当有对应类型的 surface 创建时，便会调用里面的接口。实现 PluginSurface 的创建和 wayland 合成器端的bingding。

    widget 插件将通过 客户端封装的接口来调用 实现后的request 和接受对应的 event。

    细节：
    1. 作为一个 wayland-shell-integration 插件实现
    2. 实现一个 QWaylandShellIntegrationPlugin 和 QWaylandShellIntegration 将 DockPluginManager 的创建和 PluginSurface 的创建注册到 wayland 中
    3. 定义 DockPlugin 作为 QWindow 和 shell-surafce 的桥梁。QWindow， DockPlugin 和 DockPluginSurface 一一对应。
    4. 实现 DockPluginManager
        ```c++
        class DockPluginManager : public QObject, public QtWayland::dock_plugin_manager_v1
        ```
    5. 将 event 转成对应的信号对外暴露，实现属性关联 request。
    6. 实现 createPluginSurface 接口，创建DockPluginSurface

    7. 实现 DockPluginSurface 并根据 DockPlugin 将对应的 QWindow 关连起来。
        ```c++
        class DockPluginSurface : public QtWaylandClient::QWaylandShellSurface, public QtWayland::dock_plugin_surface
        ```
    8. 将 event 转成对应的信号暴露给 DockPlugin，并将 DockPlugin 的信号关联到 DockPluginSurface 调用 request 的槽上。

3. widget 插件的接口转到实现的协议上

    widget 插件是一个旧接口对接到新协议的中间类。他通过实现 proxy 的接口来让插件认为它就是 dock 本体。又通过和 dock 插件管理协议的关联，将插件的请求通过 wayland 转发给 dock。

    在 main 中通过 PluginLoader 将插件进行加载，成功加载后构造对应的widgetplugin 对象， widgetplugin 将旧接口 pluginsiteminterface 和 pluginproxyinterface 通过 DockPlugin 接口类与协议关联起来，实现将对应的 wayland event/request 转成 dock 的接口相关。

4. dock 又是如何"抓取"插件

    插件注册到 wayland 合成器后，在dock 的 DockCompositoer 中根据 surface 中定义的 type 属性，将 surface 存入到不同的 Model 中。然后 dock 对应的部分（比如tray部分）将对应的Model（traySurfaceModel）用于 tray 区域的显示，并使用 Qt 提供的 ShellSurfaceItem 将对应的  surface 显示出来。在 dock 上的布局和排列就可以按照 shell-surface 上定义的属性来实现。

    但是对于插件的 tooltip 和 popup 处理特殊一下，主要有两种思路：

    1. tooltip 和 popup 按需提供，需要将对应的 hover 事件转到客户端，然后调用插件接口产生 tooltip/popup 的窗口出来，然后将该窗口贴到 tooltip 和 popup 上去。

    2. tooltip 和popup 上来就注册到wayland中，dock 按需将对应的 item 贴到 dock 的 tooltip/popup 上去。

    目前采用的是第二种方案，但是对于网络插件自身有判断，不能同时产生 tooltip 和 popup，两者只可存在一个，导致目前网络插件没有对应的 tooltip 产生。

## 实现后的效果

dde-shell 中插件与dock分进程，目前托盘插件正常显示和响应对应输入，tooltip和popup正常处理。

所有插件各自独立进程
![效果图](https://s2.loli.net/2024/03/18/inZueY643Cha1HE.png)

## 畅想和待做

1. 对于有输入插件（比如网络插件，输入情况的支持需要确认，应该是需要添加输入法协议的支持）

2. 不同类型的插件需要准备多种loader

待做：

1. 实现一个 QML loader 并完善 QML的支持，用于编写 QML 插件。
