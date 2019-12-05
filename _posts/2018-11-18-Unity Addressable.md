---
layout: post
title: Unity Addressable
---

Unity2018.2之后提供了一个新的Package， Addressable System。这个系统用来替代之前的AssetBundle， 提供了一系列的方法来为开发者部署游戏资源提供便利。我使用之后觉得非常的不错，这里学习一下这个系统的使用方法和设计思路以及我的一些想法。

这个包依赖于Resource Manager（RM）和Scriptable Build Pipeline（SBP）两个包。Unity在写包的时候还是非常专业的，思路非常清楚，把需求严格地分离开。RM和SBP这两个包单独都可以使用，开发者可以利用它们进行二次开发。在这一点上，我们公司地代码可以说是毫不考虑这一点了，以至于每个项目都像一个新项目一样，没有代码积累和迁移。简而言之，抄代码都抄的很不舒服。我以后写功能要多向Unity学习。

因为这个系统还在开发阶段，之后一定会不断更新，因此本文也会按照版本更新来更新。

## Addressable System v0.4.8-preview   2018.11.18

寻址系统目的是为了解决开发者在资源寻址时的各种代码书写，打包时的各种代码书写，其中很多不方便，也写不好的地方。这部分代码其实不太应该由开发者来写。比如崩3的资源打包，加载，少说也写了四五千行代码，代码质量不说，复用性很低，很hardcode。

### Address

每个资源都有一个唯一的string Address，比如目前来说，Resources目录下的资源的Address为它相对于Resources目录的path去掉后缀名。这点在Address System中保留了下来。要理解string Address只是一个代号，不管有没有"/"，有没有后缀名，在此系统中都已经没有意义了。随便你怎么安排Address.

### Asset Group

他是一组资源。它可以是AssetBundle，也可以不是。Resources下的内置资源也是一组asset group，它显然就不是AB。另外用了Address以后就不需要用AB了，它提供了兼容的办法，新项目没有这个问题，老项目的话，迁移成本太高。我感觉手游还没有这个需求或者必要。

### Asset Group Schema

定义了打包时会怎么打。目前自定义的AssetGroup都必须有Content Update Group Schema和Bundle Asset Group Schema. 这说明这些资源会被打包成AB。Bundle Asset Group Schema还定义了这个包里的资源build时打包到哪里，Load时从哪里读取。
Addressable中定义了runtime path是Application.streamingAssetsPath + "/com.unity.addressables", 这是一个默认设置。打包时把ab打包到Streaming Path里，这样打包时会直接复制到对应目录中，而不再由Unity打包。因此Addressable这里把Streaming Asset目录作为资源打包后bundle存的目录。但是注意这里是不能热更的。

#### Asset Bundle Provider Type

### Profile

这是最重要的实现多种打包方式的配置文件。我们可以添加多种配置方式。比如打包一个Windows整包的Profile, 打包一个Android整包，打包一个Android分包等等。通过Profile Entry定义了一组key/value，然后在Asset Group Schema中会用对应的key/value来打包。想打分包时只要不把资源加到Streaming Assets里即可。

### Label

提供一种额外的方式来加载资源。目前这个版本，label机制还没有实现，只是填了个坑。

### Play Mode

内置了3种Play Mode, 是为我们在开发时用的。Fast Mode加载资源都用Asset Database里加载。Virtual Mode也差不多，我测试中可以看到Virtual Mode也是从Asset Database里加载的。 Packed Mode则是模拟了真实情况，该从哪加载就从哪加载。

### 打包

Addressable目录下面的AddressablesBuildScriptHooks.cs文件定义了怎么打包。它改了一下正常的Build流程。如果项目里有Address，就用这套打包流程。我没有找到手动打包的地方，可以自己用SBP来实现。我稍微试了下，可以打出AB，然后想办法配置。

### AB下载

这里提供了一个叫Hosting的Http服务器，来给我们测试AB下载功能。根据官方文档的说明，我操作了一下。  
打包会打出数个文件  
catlog_[time].hash 对下面json文件进行检验的hash  
catlog_[time].json  
catalog_BuildScriptPackedMode.hash 对下面json文件进行检验的hash  
catalog_BuildScriptPackedMode.json 一个典型的资源json，里面存了如下内容  

``` json
{
    "m_providerIds": [
        "UnityEngine.ResourceManagement.SceneProvider",
        "UnityEngine.ResourceManagement.LegacyResourcesProvider",
        "UnityEngine.ResourceManagement.AssetBundleProvider",
        "UnityEngine.ResourceManagement.BundledAssetProvider"
    ],
    "m_internalIds": [
        "Scenes/SampleScene",
        "resed",
        "http://192.168.31.214:62298/packedassets_assets_all_a9f21d6f425317aa1406de36c06003b1.bundle",
        "Assets/Resources_moved/streaming.txt",
        "Assets/Resources_moved/addressed.txt"
    ],
    "m_keyDataString": "CgAAAAALAAAAU2FtcGxlU2NlbmUFIDk5Yzk3MjBhYjM1NmEwNjQyYTc3MWJlYTEzOTY5YTA1BAAAAAAABQAAAHJlc2VkBSA3Njk5MWM0OTYxNTBmYzU0YWFhMGM2N2UwZTg0ODVlZgAeAAAAcGFja2VkYXNzZXRzX2Fzc2V0c19hbGwuYnVuZGxlAAkAAABzdHJlYW1pbmcFIDc5MWZlMjk3ZmNjODc1MDRlYmQ2M2YxNTY5MmI4NDhiAAkAAABhZGRyZXNzZWQFIDgzZjhkM2VlODcyMjQ3NjRiOWUxNzczYTI3NGFmODdl",
    "m_bucketDataString": "CgAAAAQAAAABAAAAAAAAABQAAAABAAAAAAAAADYAAAABAAAAAAAAADsAAAABAAAAAQAAAEUAAAABAAAAAQAAAGcAAAABAAAAAgAAAIoAAAABAAAAAwAAAJgAAAABAAAAAwAAALoAAAABAAAABAAAAMgAAAABAAAABAAAAA==",
    "m_entryDataString": "BQAAAAAAAAAAAAAA//////////8BAAAAAQAAAP//////////AgAAAAIAAAD/////AAAAAAMAAAADAAAABQAAAP////8EAAAAAwAAAAUAAAD/////",
    "m_extraDataString": "BkxVbml0eS5SZXNvdXJjZU1hbmFnZXIsIFZlcnNpb249MC4wLjAuMCwgQ3VsdHVyZT1uZXV0cmFsLCBQdWJsaWNLZXlUb2tlbj1udWxsOFVuaXR5RW5naW5lLlJlc291cmNlTWFuYWdlbWVudC5Bc3NldEJ1bmRsZVJlcXVlc3RPcHRpb25zHAEAAHsAIgBtAF8AaABhAHMAaAAiADoAIgBhADkAZgAyADEAZAA2AGYANAAyADUAMwAxADcAYQBhADEANAAwADYAZABlADMANgBjADAANgAwADAAMwBiADEAIgAsACIAbQBfAGMAcgBjACIAOgAyADkANQA3ADYAMAAwADIANQA2ACwAIgBtAF8AdABpAG0AZQBvAHUAdAAiADoAMAAsACIAbQBfAGMAaAB1AG4AawBlAGQAVAByAGEAbgBzAGYAZQByACIAOgBmAGEAbABzAGUALAAiAG0AXwByAGUAZABpAHIAZQBjAHQATABpAG0AaQB0ACIAOgAtADEALAAiAG0AXwByAGUAdAByAHkAQwBvAHUAbgB0ACIAOgAwAH0A"
}
```

packedassets_assets_all_[hashid].bundle * N  hashid更新以后应该会自动下载

build player之后，确实会下载ab，然后读出其中的内容。但是奇怪的是，修改项目中的资源后，重新打ab，player并不会加载新的ab。  
（已提交BUG）经过测试，在项目中必须要至少有一个static content, 才能在打包时打出bin。这样才能build for content update.如此设置后，就可以正常地热更新了。并且static content也是可以更新的。只要prepare一下就可以了。build后，上面json中的internal id会改变，等于告知了系统需要更新这部分了。  
android打包无法使用addressable这套系统。这是0.4.8的bug，只能等之后版本看看了。  

研究至此，我认为这套系统已经可以用于实际项目中了（只要BUG修了）。Http服务器这块我还要研究一下，看看是不是有什么特别的。  

![queenlake]({{ site.baseurl }}/images/queenlake.jpg)