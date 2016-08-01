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
Nuwa项目包含4个部分：
- extras

  Hack.java，为了避免CLASS_ISPREVERIFIED错误发生，该文件是进行字节码注入时候使用的类。字节码注入待gradle插件分析时候解释。
  ```
  public class Hack {}
  ```
- nuwa/assets

  assests目录包含hack.apk文件。该文件在Nuwa.init中将被加载。文件内容如下：
  ![](https://github.com/lihenair/Read-note/blob/master/image/hack_apk_content.png)
  
- nuwa/java
  将hack.apk载入到application中的方法是combineArray，该函数也用于加载patch.jar文件。调用栈如下：
  ![](https://github.com/lihenair/Read-note/blob/master/image/combine_array_call_hierarchy.png)
  ```
  private static Object combineArray(Object firstArray, Object secondArray) {
        //获取新数组对象
        Class<?> localClass = firstArray.getClass().getComponentType();
        //获取新数组中的长度
        int firstArrayLength = Array.getLength(firstArray);
        //计算合并后的数组长度
        int allLength = firstArrayLength + Array.getLength(secondArray);
        //构造新的数组
        Object result = Array.newInstance(localClass, allLength);
        //合并新旧数组
        for (int k = 0; k < allLength; ++k) {
            if (k < firstArrayLength) {
                Array.set(result, k, Array.get(firstArray, k));
            } else {
                Array.set(result, k, Array.get(secondArray, k - firstArrayLength));
            }
        }
        return result;
    }
  ```
  该函数首先将需要加载的类计算出个数，之后加上旧的类个数，然后合并到新的array对象中。
  
  注入到Dex头的方法是injectDexAtFirst。该方法首先获取默认的DexClassLoader，之后分别获取ClassLoader的pathList对象中的dexElements对象，该对象包含class的数据。在查找类时候会从dex文件开始，遍历dexElements数组中的类。找到后就退出，不在继续查找。那么injectDexAtFirst负责就是处理这个合并的操作，并将合并后的dexElements数据写回到系统的DexClassLoader中。
  ```
  public static void injectDexAtFirst(String dexPath, String defaultDexOptPath) throws NoSuchFieldException, IllegalAccessException, ClassNotFoundException {
        //构造系统的DexClassLoader
        DexClassLoader dexClassLoader = new DexClassLoader(dexPath, defaultDexOptPath, dexPath, getPathClassLoader());
        //获取当前的dexElements数组
        Object baseDexElements = getDexElements(getPathList(getPathClassLoader()));
        //获取新的dex/jar中的dexElements数组
        Object newDexElements = getDexElements(getPathList(dexClassLoader));
        //合并两个dexElements
        Object allDexElements = combineArray(newDexElements, baseDexElements);
        //得到当前DexClassLoader的pathList对象
        Object pathList = getPathList(getPathClassLoader());
        //将新生成的dexElements数组写回到系统的DexClassLoader中
        ReflectionUtils.setField(pathList, pathList.getClass(), "dexElements", allDexElements);
    }
  ```

至此，Nuwa的分析结束。

###NuwaGradle分析
本段分析NuwaGradle

参考链接：[安卓App热补丁动态修复技术介绍](https://mp.weixin.qq.com/s?__biz=MzI1MTA1MzM2Nw==&mid=400118620&idx=1&sn=b4fdd5055731290eef12ad0d17f39d4a&scene=1&srcid=1106Imu9ZgwybID13e7y2nEi#wechat_redirect)
