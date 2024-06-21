---
title: AB 包学习笔记
tags:
---


# AB包
- 配置：在`AutoBuild.cs`脚本的`SetAB()`方法中配置需要打包的文件夹
- 打包：
	- 工具栏中`Huya -> AutoBuild -> BuildPC64 AB` 打AB包，可选是否加密
	- `Huya -> AutoBuild -> BuildPC64 `打PC包，会打到`ProjectName/Build`文件夹下

### 相关链接
[腾讯云AB包详解](https://cloud.tencent.com/developer/article/2097273)

### AB包原理
刚好有空剑哥跟我仔细讲了一下我们这边AB包的技术方案和一些细节。
一共两套方案，一套现在车车用的，一套之前做多游戏项目做的。
#### AB打包

#### AB分组策略
##### 按逻辑实体分组
适用于可下载内容（DLC），对于增量或指定实体的更新比较方便，具体策略为：
- 一个或所有UI界面->一个AB包（包含界面自身prefab，引用的贴图，材质等资源等），需要更新UI界面表现时方便。
- 一个角色或所有角色->一个AB包（包含角色的模型贴图材质资源，动画资源等），有新角色更新，或者角色调整时方便。
- 所有场景共享的部分一个包（贴图模型等）

假设一个项目有新需求，要新出了一个副本地图，有新的怪，同时UI界面的画面风格配合新副本有所调整，则AB包的更新策略为：
- 增加一个新场景的AB包（依赖于场景共享AB）
- 更新包含全角色的AB包或为每个新怪添加一个新AB包
- 更新包含全UI的AB包或更新需要调整画风的个别UI的AB包

##### 按资源类型分组
按照资源的格式与类型分组，适用于有多平台发布需求的项目，具体策略举例：
- 项目需要发布到windows和mac平台，音频数据在两个平台是通用的，则所有音频打到一个AB包中。而shader资源需要根据不同平台进行适配，则适用于windows的shader打到一个AB包，适用于mac的打到另一个AB包。发布后根据平台来选择性加载对应的AB包。模型资源，材质资源同理。
- 除此之外，这种策略可以提高对不同Unity版本的兼容性，因为纹理的包在不同版本Unity中基本都通用（纹理的压缩格式和设置的变化频率较低，代码/预制体较高）。

##### 按照使用分组
从使用时机、使用频率的角度可以制定不同的策略。
- 将需要同时加载的资源“捆绑”在一起，打成一个AB包，例如：游戏有多个场景（关卡），则将每个场景（关卡）的所有地图角色模型，材质，音频资源等全部打到一个AB包中，便于在切换场景（关卡）时一次性完成所有所需AB包的加载。
- 将使用频率接近的资源打到同一包中，例如：
	- 一个包中只有50%的资源经常同时加载，则拆分这个包
	- 两组不可能同时加载的资源（如贴图的标清版本与高清版本），则拆分为两个包
	- 共享资源打到一个包，例如不同怪物npc共享的动画资源；不同模型共享的体贴材质资源




#### Manifest文件
每个AB包包含一个.ab文件和一个同名的.manifest的清单文件。具体的结构如下
```javascript
ManifestFileVersion: 0
CRC: 4225903359
AssetBundleManifest:
  AssetBundleInfos:
    Info_0:
      Name: share.u3d
      Dependencies: {}
    Info_1:
      Name: cubewall.u3d
      Dependencies:
        Dependency_0: share.u3d
    Info_2:
      Name: spherewall.u3d
      Dependencies:
        Dependency_0: share.u3d 
```
其中循环冗余校验码（CRC）用于校验AB包的完整性，AssetBundleInfos中包含了AB包的名字和依赖关系。


## 项目方案

#### 方案一（现在用的）
PC.manifest作为所有AB的跟配置信息文件，其余AB分各自的manifest。用的是Unity自带的BuildAseetBundle方法来打包。
加载时，首先加载PC.manifest，然后根据PC.mainfest获取所有的AB包的路径和对应的依赖包。读取到Dic中（保留在内存中）。随后需要加载AB包中的资源的时候，根据AB自己的manifest递归的加载所有依赖的AB包。当需要AB包a中的某个资源，例如prefab时，就会加载整个该ab中的资源到内存，然后卸载ab包（好像是，有点忘记了寄，得再看下代码）
##### 打包
- `BuildPlayer.cs`中包含了打包到不同平台的方法，本质是对Unity提供的`BuildPipeline.BuildAssetBundle()`方法的封装。
	- BuildPipeline.BuildAssetBundle()中传入了4个参数，返回打包的结果为AssetBundleManifest
		```C#
		// outputPath: AB包输出路径
		// assetBundleArray: 需要打的AB包的名字集合
		// buildOption: 一些配置参数，例如压缩规范等
		// buildTarget: 打包的目标平台
		AssetBundleManifest manifest = BuildPipeline.BuildAssetBundles(outputPath, assetBundleArray, buildOption, buildTarget);
		```
##### 加载
#### 方案二（多游戏项目用的）
自己设计的ABConfig文件，不适用manifest，自己定义的json格式来解析。按需加载AB包，同样是全部AB包的name作为key，资源作为value
