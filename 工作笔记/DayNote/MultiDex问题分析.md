## MultiDex问题分析
### 简单描述MultiDex
大概就是Dalvik在优化中，认为一个方法使用到的寄存器范围在8-16个之间，能handle的Method、Field的数量在2^16个之内。
所以为了保证每个dex文件能被Dalvik正确处理，在打包过程中，dx工具会对所有的class文件遍历，保证每个dex文件的所有索引数目在64k以内。
### 实现MultiDex
为了支持多dex文件在Dalvik中运行，Google提供了MultiDex支持库。而在5.0以后，Android支持了ART运行时，它会在应用安装时候执行预编译，它会把所有的dex文件合并编译成一个oat文件，从而实现了原生支持多dex的功能。
### MultiDex实现原理简介
`Application`中的`attachBaseContext`是我们能控制到的应用的最早执行的代码，在其中调用`MultiDex.install()`方法，这个方法会去检索当前机器的版本，在5.0以下的版本中，会去加载其余的dex文件。
### MultiDex带来的问题
#### ClassNotFound
在Framework调起功能的时候，如果使用到的代码存在于未被加载的dex中，Dalvik会视为ClassNotFound。
#### 冷启动缓慢
第一次启动应用时，Dalvik虚拟机在加载dex文件前会使用dexopt工具对dex文件进行优化为odex文件再去执行。
#### OOM
加载副dex时需要大内存。
这个存在于很垃圾的手机当中，内存太小，可以不处理。
### 针对上面的各种问题采用的处理方式
#### ClassNotFound
把所有可能在应用启动阶段使用到的类打包在主dex文件中
#### 冷启动缓慢
1. 主dex包变小，易于dalvik加载
2. 在`MultiDex.install()`中使用到了很多IO操作，异步调用加载可以提高加载速度。
### Google支持库中Gradle所做的处理
#### 针对ClassNotFound
在gradle的编译过程中会有专门对MultiDex的处理的task
在1.4以下的gradle版本中，会有三个task
1. collectDebugMultiDexComponents
2. shrinkDebugMultiDexComponents
3. createDebugMainDexClassList
功能依次是把`AndroidManifest`中所有相关类存储到`manifest_keep.txt`中；根据proguard以及我们添加的keep文件以及上一步生成的keep文件中的类来筛选一遍需要放到主dex中的类，并生成一个含有所有class的`componentClasses.jar`文件；遍历`componentClasses.jar`中的所有类，利用类似Proguard的原理来寻找最早使用的类，把这些类写进`maindexlist.txt`文件。
`manifest_keep.txt`，`componentClasses.jar`，`maindexlist.txt`这三个文件存放在`build/intermediates/multi-dex/debug`目录下，而`maindexlist.txt`中记录的类文件会打入主dex中。
在1.4以后的三个beta版本中

![](./_image/2017-11-14-15-59-22.jpg)
隐藏掉了这三个task，并把这几个功能分散到了编译过程中：

![](./_image/2017-11-14-16-02-44.jpg)

![](./_image/2017-11-14-16-02-52.jpg)
### 为什么google处理了我们还会存在这些
1. 反射调用到的类，脚本是不会把这些类添加到`maindexlist.txt`中
2. 脚本本身的问题，可能会漏掉...... 猜测可能是添加到`maindexlist.txt`的类冗余导致没有办法去添加必要的class， 也有可能是field超过了64k而导致分到主dex的class不充分
### 解决办法
#### ClassNotFound
1.4以下的gradle版本内可以打钩子写脚本去寻找需要放进主dex中的class，去替换`maindexlist.txt`中的内容，而1.4以后的版本在gradle中去做这些事情是被禁止的，会报出

```
Error:(32, 0) Access to the dex task is now impossible, starting with 1.4.0
1.4.0 introduces a new Transform API allowing manipulation of the .class files.
See more information: http://tools.android.com/tech-docs/new-build-system/transform-api
Open File
```

所以在1.4版本以上去写替换`maindexlist.txt`的脚本还没有研究出来怎么做...

大部分项目的做法就是经过大量测试，把可能出现问题的class，去配置一个proguard文件去keep住需要放在主dex的。

#### 冷启动缓慢
由MultiDex导致的冷启动缓慢只会发生在5.0以下的版本中
在ART的环境下，apk在安装过后会预编译成oat文件
所以不存在分包加载的问题

在5.0以下启动缓慢是由于应用在第一次启动的时候dexopt会把dex文件编译成odex文件，这个过程是主要耗时的部分
而这个有分成两个部分
1. 主dex庞大
2. 加载副dex需要时间

##### 主dex庞大
在1.4以下可以打钩子去添加设置
在2.2以上可以配置dexOption
而debug版本，在设置了一个dex的方法为40000时是编译失败的，具体原因还在看。
设置成50000是可以编译通过，dex文件的数量变多一个 3 -> 4
但是在4.2华为手机上测试并没有多大的改变，还是很慢
##### 副dex加载
异步调用`MultiDex.install()`，复写`Instrumentation`，修改`execStartActivity`与`newActivity`方法，在未加载完成所有副dex的情况下跳转到一个特定的展示页面，加载完成后再去进入功能。
这个方案是美团提供，具体实现还需要调研。
### 在6.0手机上的冷启动问题
查看日志像是oat文件有错误，重新去加载dex
```
Fallback to original dex file with interpret-mode for /data/app/com.kingsoft.email-2/split_lib_slice_3_apk.apk
```
这个问题由于无法复现，猜测是在安装过程中dex文件转换成oat过程出错。

### 添加Flavor加快本地debug版本的编译速度
因为MultiDex的编译过程需要去执行而外的task寻找需要放在主dex的class，而在21+的手机上是不存在MultiDex问题的，所以可以本地添加一个minSdkVersion为21的Flavor去编译调试可以节约时间....
![](./_image/2017-11-14-19-47-35.jpg)


















