#从Calendar出发，提供本地数据以及网络数据同步的帐号服务机制
##ContentProvider
Android中管理对数据化结构集的访问。在Android基于组件的体系当中，ContentProvider很好的实现了数据的复用共享，是连接一个进程中的数据与另一个进程中运行的代码的标准界面
###Base Of ContentProvider
内容提供程序管理对中央数据存储库的访问。提供程序是 Android 应用的一部分，通常提供自己的 UI 来使用数据。 但是，内容提供程序主要旨在供其他应用使用，这些应用使用提供程序客户端对象来访问提供程序。 提供程序与提供程序客户端共同提供一致的标准数据界面，该界面还可处理跨进程通信并保护数据访问的安全性。
###关键类
* ContentResolver
* ContentProvider
* ContentObserver
* Cursor
* Uri
###Api
```java
sad
```
##LoaderManager
##SyncAdapter