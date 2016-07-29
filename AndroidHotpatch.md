##Android热修复学习——Nuwa

###学习原因
最近发版，出去之后发现了一些问题。这可咋办？覆水难收啊。看网上有完整的热修复方案，之前虽然知道，但是一直都没有深入的学习。趁此机会完整的学习一下。

###学习对象
经过搜索发现大部分都是基于Nuwa的方案。那么就以Nuwa为分析对象。

###学习准备
- Gradle
  
  包括基本知识，构建过程，插件编写等
- Groovy
  
  Gradle构建系统的基础，整个Gradle都是使用Groovy写的。是基于JVM的语言，可直接使用Java库。
- ASM
  
  在插入字节码时候会用到这个知识，也就是java编译之后生成的字节码。但是要插入到.class文件中，则需要使用ASM插入。具体介绍详见[http://asm.ow2.org/](http://asm.ow2.org/)。
- Android Classloader机制

  patch加载时候使用。
- aapt生成dex过程

  了解整个dex生成过程。
  
###Nuwa组成
分为两个git仓库
- [Nuwa](https://github.com/jasonross/Nuwa)

  该git包括patch的加载和替换。首先将asset目录中的hack.apk拷贝到data/data/packagename目录，类似的可以做到网络下载。之后是使用classloader将patch加载到dexelements最开始的位置。
- [NuwaGradle](https://github.com/jasonross/NuwaGradle)

  该git包括Nuwa的Gradle插件。插件的功能包括生成dex，字节码插入。

###Nuwa分析
本段分析Nuwa

###NuwaGradle分析
本段分析NuwaGradle

参考链接：[安卓App热补丁动态修复技术介绍](https://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a&scene=1&srcid=1106Imu9ZgwybID13e7y2nEi#wechat_redirect)
