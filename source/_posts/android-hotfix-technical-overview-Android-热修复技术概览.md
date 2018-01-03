---
title: Android热修复技术概览
date: 2017-12-27 21:30:15
tags: [hotfix, Android]
---

## hotfix比较成熟的原理
热修复需要完成的功能：代码修复，资源修复，so修复  
代码修复大体分为两类，对Java层的修改，和对native层的修改。  
已有的方向：
### Dexposed
在native层中找到需要修复的Java函数对应的mothed对象，修改为native方法，再将nativeFunc指向框架的native回调方法，再使用该方法回调Java层已经修复的方法。  
最大的问题：不支持ART虚拟机

### Native Hook
在JNI层对方法进行替换  
优点：即时生效  
缺点：无法在类中替换和删减字段，无法增加方法  

### ClassLoader替换
将修复过的类打包到dex中，插入到类加载器加载队列的最前面  
优点：以类的粒度进行修复，新增方法和字段都可以支持  
缺点：因为要加载dex才能生效，所以必须重启

### 类似Instant run
编译阶段给每个类都注入了代理变量，在执行方法时，先根据补丁包里的内容判断这个变量是否为空，如果不为空，则执行代理里的方法，为空，执行原来的逻辑  
优点：实时生效  
缺点：so和资源无法替换，侵入编译过程，对方法数，包体积都有影响，proguard的优化，对补丁的修复效果也有影响 

## 一些具体的实现
### 阿里AndFix
使用Native Hook的方式实现，获取被修复方法在native层的方法指针，将patch方法的所有native层的数据结构替换被修复方法的数据结构，以此实现方法的修复。由于这些数据结构没有缓存，下次调用时会立即使用替换过的方法的成员，因此补丁可以立即生效。
优点：补丁应用立即生效
缺点：某些情况无法进行修复，不支持新增静态变量和方法

### 腾讯QQ空间
ClassLoader方式实现。
将修复的patch构成的Elements对象插入到pathList的第一位，ClassLoader会先加载已经修复的类，忽略被修复的类，以此完成修复。  
在apk安装时，dalvik虚拟机会将dex优化成odex，优化过程中会对所有class进行校验，如果A类在static,private,构造方法，override方法中**直接引用**到B类，且A,B类在一个dex中，那么A类就会被打上CALSS_ISPERVERIFY标记。  
缺点：需要插桩消去CLASS_ISPERVERIFY，影响运行效率

### 微信Tinker
同QQ空间的方案类似，不同的是，Tinker下发的补丁通过自研的DexDiff算进行处理，体积很小。在本地后台与原应用的dex进行合并，生成全量dex，然后将dex通过hook插入到pathList中。  
优点：补丁小    
缺点：后台和成dex，如果数据包过大，会耗费很长时间，占用大量内存。  

### 美团Robust
在编译时在每个方法的开头注入一段逻辑，并在每一个类中通过注入增加一个字段，类型为一个需要补丁类实现的接口，通过判断该字段是否为空来决定是否需要加载补丁的逻辑。  
补丁需要实现上述字段的接口类型，以及一个包含所有补丁内容和需要被修复内容的类。当补丁被应用时，将反射生成包含所有补丁内容的类的实例，通过该实例获取所有补丁内容，然后根据这些内容实例化补丁类，并通过反射将这些实例设置为被修复类的字段的值。当程序执行到被修复的逻辑时，此时注入的字段为空，走补丁内容的逻辑。  
制作补丁需要额外的实现，并且，使用被修复类的成员函数或者成员变量只能通过反射来实现。

优点：
1. 正常使用ClassLoader，不涉及对系统API的反射调用，不需要针对每个版本和系统机型进行适配  
2. 补丁应用立即生效  

缺点：
1. 需要针对每一个修复的方法制作补丁，可以通过自动化制作补丁来完成，但是对开发不透明，需要通过注解或者方法调用告知自动化补丁插件，哪些方法需要添加。

### 饿了么Amigo
在框架的Application中通过反射使用自己的ClassLoader替换掉LoadedApk类中的ClassLoader。

```Java
ActivityThread activityThread = ActivityThread.currentActivityThread();
ArrayMap<String, WeakReference<LoadedApk>> packages = activity.mPackages;
LoadedApk loadedApk = packages.get(s).get();//获取第一个不为空对象
loadedApk.mClassLoader = AmigoClassLoader.newInstance(/*初始化参数*/);
```

初始化AmigoClassLoader(extends DexClassLoader)的参数为patch的路径，Amigo接受的patch文件一般为.apk，.dex和.jar也可以。
实际上也是一种全量替换，dex的加载完全交给ClassLoader，无需hook dexPathList插入Element

优点：可以新增Activity，替换so和资源文件，无需插桩
缺点：全量加载，patch文件较大

### RocooFix/HotFix/Nuwa
类似QQ空间的解决方案，含有gradle生成补丁包的插件，生成补丁没有实现完全自动化。

### 阿里Sophix
可以使用两种修复方式：  
热启动：  
类似Andfix，在native层修改方法所有的变量，指向被fix的方法，并且加强了兼容性  
冷启动：  
类全量技术，通过对dex文件的处理，把dex文件中需要patch的类文件的classRef移除，使虚拟机在处理时找不到旧dex的类文件，被patch的类都在patch.dex中，解决了dalvik虚拟机的CLASS_ISPREVERIFY的问题。

## 需要解决的问题
### 什么时候插入dex(Element)
`attachBaseContext`。Application是app进程运行时第一个被加载的类，在attachBaseContext中，Application才真正被接入AMS中，因此在这个方法中替换dex，可以最大程度减少其他类的加载

### Dlavik and ART
Dlavik虚拟机中的pre-verify
### 完整性校验，签名校验
### 生成补丁包
对开发透明：
1. 使用new.apk和old.apk进行dif，本地从服务器拉取差分文件，并在本地合并（全量）
2. 使用gradle插件，在打包时，对每个文件进行SHA1并保存，如果SHA1与上次不同，则认为是被改动了，将这个类的子类和引用该类的文件，都打包进patch中。
3. 使用new.apk和old.apk通过dex2jar进行反编译，通过解析得到的数据结构进行比较。由于是从数据结构上的比较，能够更准确地发现被改动的内容。在得到被改动的类之后，遍历所有class文件，找到直接调用他们的类，也打包进入patch。
对开发不透明：
1. 使用注解标记改动或新增的类或方法，编译时使用注解解析器将被改动的类拿出并打包进patch中。
2. (Nuwa)把被改动的类手动提取出来，由插件检测相关联的类，并一起打包进入patch


### Android 7.0+ classtable 问题
在ART虚拟机中执行的是经过dex2aot转换后的oat文件，但是dex转成oat的过程导致APP安装时间变长，转换后的oat文件比原来的dex文件大了很多，占用了太多储存空间。  
因此Android7.0+引入了[混合编译](https://github.com/WeMobileDev/article/blob/master/Android_N%E6%B7%B7%E5%90%88%E7%BC%96%E8%AF%91%E4%B8%8E%E5%AF%B9%E7%83%AD%E8%A1%A5%E4%B8%81%E5%BD%B1%E5%93%8D%E8%A7%A3%E6%9E%90.md)，安装的时候没有全部转换为机器码，而是根据运行时分析热区代码，将热区代码转换成机器码并缓存起来。在Android7.0中，每个APP都会有自己的art文件，用来缓存一些热区代码。Android7.0的ClassLoader类相比之前多了classtable属性，APP启动的时候就会将缓存的类注入到ClassLoader的classtable中，而classtable的加载是在dex之前的，要解决该问题的一个思路是替换整个ClassLoader，不走原本的缓存逻辑。


### Instant Run具体原理(gradle 2.0.0)
修改了AndroidMenifest.xml的字节码，将Application节点的android:name修改为Instant run相关的`BootstrapApplication`。  
使用transform API修改类字节码，给所有的类加上$change字段，$change为IncrementalChange类型。如果$change不为空，则调用$change的access$dispatch方法，参数为方法签名字符串和方法参数数组。  
在attachBaseContext中，通过反射，将现有ClassLoader的parent设置为`IncrementalClassLoader`
在onCreate中，将realApplication替换回应用，并执行相应的生命周期。




## 方案选择：
### 纯instant run
up: 
- 即时生效
- 兼容性强（正常使用ClassLoader，不涉及反射调用系统api）

down:
- 生成补丁复杂（反射调用原有类的成员变量），自动化补丁方式困难
- 会增加额外方法
- 侵入开发过程
- 影响编译时间

### native hook+classLoader replace
up:
- 大多数情况及时生效
- 重启生效的情况稳定性很强

down:
- 需要根据修改内容确定采取的方案
- 动态生效需要考虑的情况非常多
- dex2jar的已知问题，可能需要在开发过程中打包
- 插入代码需要根据版本源码适配
