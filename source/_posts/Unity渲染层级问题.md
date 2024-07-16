---
title: Unity渲染层级问题
comments: true
date: 2024-07-16 14:32:04
updated:
tags:
categories:
keywords:
description:
top_img:
cover:
---
# 前言

最近做2D的游戏项目比较多，有比较多需要在UI的Canvas下渲染的交互界面，粒子特效等。为了让不同层UI和粒子特效之间有正确的遮挡关系，笔者在Sorting Layer，Order in Layer，RenderQueue这些上踩了不少坑。因此自己在Unity测试了一下后，打算整理出来，一是记录，而是分享。

> 注意：以下讨论仅围绕Canvas下的渲染顺序

## 1. 纯UI

在无自定义Shader，使用Unity默认UI材质的情况下（不考虑RenderQueue）。渲染受Canvas组件中Sorting Layer和Order in Layer参数的影响。下面分几种场景情况

### 1.1 同Canvas

同一Canvas下，渲染顺序受UI物体在Hierarchy的位置的影响，遵循”下压上“的原则。即在Hierarchy中靠下的物体，在渲染时会压住同一层级中上面的物体。举例如下：

![](https://jasper-cos-1320024929.cos.ap-guangzhou.myqcloud.com/PicGo/20240716155826.png)

如上，红、黄、绿、蓝共4个Image都在Canvas1下。Hierarchy中从下到上是蓝->绿->黄->红，根据“下压上”的原则，可以在Game窗口看到确实是蓝压着绿压着黄压着红（会不会写的太细了...）。

### 1.2 不同Canvas

涉及多个Canvas下的UI物体渲染顺序比较时，就需要考虑Sorting Layer和Order in Layer这两个参数。按笔者理解，Sorting Layer是Unity提供的用于开发者自定义渲染层。

**多Canvas时，不同Canvas的渲染先比较Sorting Layer的优先级，同一Sorting Layer的情况下再比较Order in Layer。**

Unity默认提供一个Default层，可以通过下图方式添加自定义层。

![](https://jasper-cos-1320024929.cos.ap-guangzhou.myqcloud.com/PicGo/20240716173420.png)

通过拖动目标Layer所在行就可以调节这个Sorting Layers列表中不同Layer的顺序。

![](https://jasper-cos-1320024929.cos.ap-guangzhou.myqcloud.com/PicGo/20240716161119.png)

这里的顺序就决定了Layer的优先级。类似上面的"下压上"，在Sorting Layers中，越靠下的Layer，渲染的优先级越高。如下：

![](https://jasper-cos-1320024929.cos.ap-guangzhou.myqcloud.com/PicGo/20240716161531.png)

笔者当前的Sorting Layers设置如上，此时Layer的优先级应该是Default > New Layer 1。

回到场景中，Canvas1属于Default，

![](https://jasper-cos-1320024929.cos.ap-guangzhou.myqcloud.com/PicGo/20240716173918.png)

Canvas2属于New Layer 1

![](https://jasper-cos-1320024929.cos.ap-guangzhou.myqcloud.com/PicGo/20240716173946.png)

由于Default优先级高于New Layer 1，Canvas1的红色压着Canvas2的蓝色（尽管Canvas2的Order in Layer = 1000 大于Canvas1，但先比较Sorting Layer）

接下来看如果Sorting Layer相同

![](https://jasper-cos-1320024929.cos.ap-guangzhou.myqcloud.com/PicGo/20240716174244.png)

如上Canvas3跟Canvas2的Sorting Layer一样都是New Layer 1。但Canvas2(Order in Layer = 1000)> Canvas3(Order in Layer = 500)，所以蓝色压着绿色。


## 2. 粒子

**粒子虽然挂在Canvas下，但渲染顺序不受其父节点上Canvas组件的SortingLayer和OrderinLayer影响。**

粒子的渲染优先级受Particle System中Render里的Sorting Layer 和 Order In Layer控制

![](https://jasper-cos-1320024929.cos.ap-guangzhou.myqcloud.com/PicGo/20240716175223.png)

粒子之间的渲染优先级与纯UI相同，先比较Sorting Layer，再比较Order in Layer。但笔者自己测试观察发现，Sorting Layer 和Order in Layer 都相同时，并未发现和纯UI一样“下压上“的规则，具体规则暂时不清楚。


## 3. RenderQueue

这部分主要参考这篇博客[《Shader中的RenderQueue》](https://blog.csdn.net/linxinfa/article/details/105361396#:~:text=in%20Layer%E3%80%82-,%E5%9B%9B.%20Shader%E7%9A%84RenderQueue)

RenderQueue是Shader里的渲染队列值，Unity内部对不同的RenderQueue值有明确的划分
	![image.png](https://jasper-cos-1320024929.cos.ap-guangzhou.myqcloud.com/PicGo/20240716111039.png)
主要注意2500是一个关键划分值，在Shader中是透明跟不透明的分界点。无论SortingLayer和OrderInLayer的值，`RenderQueue > 2500`的物体必定会在 `RenderQueue <= 2500`的物体前面，即划分在“透明区”（>2500）的肯定会压着“不透明区”（<=2500）的。

当物体都在同一“区”时，排序优先级为Sorting Layer > Order in Layer > RenderQueue
