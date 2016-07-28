#Gradle实践笔记#

##Chapter 3 Gradle简单构建

####使用java插件
每个Gradle工程都有一个build.gradle做为编译入口
编译java工程要引入java的plugin

```
apply plugin: 'java'
```
####修改工程和插件属性

```
version = 0.
sourceCompatibility = 1.6
jar {
    manifest {
        attributes 'Main-Class': 'com.manning.gia.todo.ToDoApp'
    }
}
```
####适配工程结构

```
sourceSets {
    main {
        java {
            srcDirs = ['src']
        }
    }
    test {
        java {
             srcDirs = ['test']
        }
    }
}
buildDir = 'out'
```
####定义仓库

```
repositories {
    mavenCentral()
}

####定义依赖
dependencies {
    compile group: 'org.apache.commons', name: 'commons-lang3', version: '3.1'
}
```

####获取依赖
下载依赖的task是

```
$ gradle build
:compileJava
Download http://repo1.maven.org/maven2/org/apache/commons/commons-lang3/3.1/commons-lang3-3.1.pom
Download http://repo1.maven.org/maven2/org/apache/commons/commons-parent/22/commons-parent-22.pom
Download http://repo1.maven.org/maven2/org/apache/apache/9/apache-9.pom
Download http://repo1.maven.org/maven2/org/apache/commons/commons-lang3/3.1/commons-lang3-3.1.jar
```

####provide和runtime
providedCompile用于编译时 但运行时由运行环境提供的依赖 因此依赖不会被打包到生成的最终文件里
runtime不需要在编译时处理 但运行时需要 因此将被打包到最终文件里

```
application:dependencies
{    
    providedCompile 'javax.servlet:servlet-api:2.5'
    runtime 'javax.servlet:jstl:1.1.2'
}
```

####设置wrapper
设置task的类型为Wrapper，gradle版本是gradleVersion

```
task wrapper(type: Wrapper) {
    gradleVersion = '1.7'
}
```
gradle wrapper只需执行一次，执行后将下载如下文件

```
├── gradle
│     └── wrapper
│            ├── gradle-wrapper.jar
│            └── gradle-wrapper.properties
├── gradlew
└── gradlew.bat
```
```
task wrapper(type: Wrapper) 
{
    gradleVersion = '1.2'
    distributionUrl = 'http://myenterprise.com/gradle/dists'
    distributionPath = 'gradle-dists'
}
```
##Chapter 4 构建脚本要领

###构建块
project，task， properties

```
task printVersion(group: 'versioning', description: 'Prints project version.') << {
   logger.quiet "Version: $version"
}

task backupReleaseDistribution(type: Copy) {
    from createDistribution.outputs.files
    into "$buildDir/backup"
    }

task release(dependsOn: backupReleaseDistribution) {
    logger.quiet 'Releaseing the project...'
    }
```

###声明任务的输入和输出
Gradle以task的input和output是否有变化来确定task是否是**UP-TO-DATE**。如果没有变化，那么task就是**UP-TO-DATE**的。因此，只有输入或输出不同了，task才会执行。

###Custom task
####定义任务

```
class ReleaseVersionTask extends DefaultTask {
    @Input Boolean release
    @OutputFile File destFile

    ReleaseVersionTask() {
        group = 'versioning'
        description = 'Makes project a release version.'
    }

    @TaskAction
    void start() {
        project.version.release = true
        ant.propertyfile(file: destFile) {
            entry(key: 'release', type: 'string', operation: '=', value: 'true')
        }
    }
}
```
####使用任务

```
task makeReleaseVersion(type: ReleaseVersionTask) {
   release = version.release
    destFile = versionFile
    Setting custom task properties
    Defining an enhanced task of type ReleaseVersionTask
}
```
####task type

```
task createDistribution(type: Zip, dependsOn: makeReleaseVersion) {
    from jar.outputs.files
    
    from(sourceSets*.allSource) {
        into 'src
    }
    
    from(rootDir) {
        include versionFile.name
    }
}
```

###task rule

```
task incrementMajorVersion(group: 'versioning', description: 'Increments project major version.') << {
    String currentVersion = version.toString()
    ++version.major
    String newVersion = version.toString()
    logger.info "Incrementing major project version: $currentVersion -> $newVersion"
    ant.propertyfile(file: versionFile) {
        entry(key: 'major', type: 'int', operation: '+', value: 1)
    }
}

task incrementMinorVersion(group: 'versioning', description: 'Increments project minor version.') << {
    String currentVersion = version.toString()
    ++version.minor
    String newVersion = version.toString()
    logger.info "Incrementing minor project version: $currentVersion -> $newVersion"
    ant.propertyfile(file: versionFile) {
		entry(key: 'minor', type: 'int', operation: '+', value: 1)
    }
}					
```


####Building code in buildSrc dircetory

buildSrc是构建代码可选的放置地。Groovy的目录结构是src/main/groovy。任何该目录中的代码都会自动编译并放入到Gradle构建脚本的classpath。

```
├── build.gradle
├── buildSrc
|      └── src
|           └── main
|                 └── groovy 
|                        └── com
|                             └── manning
|                                    └── gia
|										  ├── ProjectVersion.groovy
|										  └── ReleaseVersionTask.groovy
├── src
│    └── ...
└── version.properties
```

####Custom task in code

```
package com.manning.gia
import org.gradle.api.DefaultTask
import org.gradle.api.tasks.Input

import org.gradle.api.tasks.OutputFile
import org.gradle.api.tasks.TaskAction
class ReleaseVersionTask extends DefaultTask {
    (...)
}
```

####Hooking into the task execution graph

![](https://github.com/lihenair/Read-note/blob/master/image/build_lifecycle_hocks.png)

```
gradle.taskGraph.whenReady { TaskExecutionGraph taskGraph ->
    if(taskGraph.hasTask(release)) {
        if(!version.release) {
            version.release = true
            ant.propertyfile(file: versionFile) {
                entry(key: 'release', type: 'string', operation: '=', ➥ value: 'true')
            }
        }
   }
}
```

####实现任务执行图监听器

![](https://github.com/lihenair/Read-note/blob/master/image/task_execution_graph_listener.png)

```
package com.manning.gia

import org.gradle.api.execution.TaskExecutionGraph
import org.gradle.api.execution.TaskExecutionGraphListener

class ReleaseVersionListener implements TaskExecutionGraphListener {
    final static String releaseTaskPath = ':release'

    @Override
    void graphPopulated(TaskExecutionGraph taskExecutionGraph) {
        if (taskGraph.hasTask(releaseTaskPath)) {
            List<Task> allTasks = taskGraph.allTasks
            Task releaseTask = allTasks.find { it.path == releaseTaskPath }
            Project project = releaseTask.project

            if (!project.version.release) {
                project.version.release = true
                project.ant.propertyfile(file: project.versionFile) {
                    entry(key: 'release', type: 'string', operation: '=', value: 'true')
                }
            }
        }
    }
}
```

##Chapter 5 依赖管理

###依赖管理例子
使用dependencies引入依赖的包，repositories设置从何地获取依赖。
![](https://github.com/lihenair/Read-note/blob/master/image/gradle_load_dependency.png)

###依赖配置
![](https://github.com/lihenair/Read-note/blob/master/image/comfiguration_api_in_gradle.png)

Java插件提供了6个配置:compile, runtime, testCompile, testRuntime, archives, 和default。

###自定义配置

```
configurations {
    cargo {
        description = 'Classpath for Cargo Ant tasks.'
        visible = false
    }
}
```
查看增加的配置

```
$ gradle dependencies
:dependencies
------------------------------------------------------------
Root project
------------------------------------------------------------
cargo - Classpath for Cargo Ant tasks.
No dependencies
```
增加配置后，就可以直接访问配置名称了。

###配置UML
![](https://github.com/lihenair/Read-note/blob/master/image/dependencies_uml.png)

每个以来都是一个Dependcy实例，包括group,name,version和classifier属性。

###剔除依赖
```
dependencies {
    compile('org.codehaus.cargo:cargo-ant:1.3.1') {
        exclude group: 'xml-apis', module: 'xml-apis'
        transitive = false
    }
}
```

exclude方法可以剔除传递依赖，使用group或module指定依赖名称。Gradle不允许剔除指定版本，所以verison不会出现。
transitive属性用于控制依赖的传递性。如果设置为false，则禁用了这个依赖的所有传递依赖。

###动态版本

```
dependencies {
    cargo 'org.codehaus.cargo:cargo:cargo-ant:1.+'
}
````
如果想使用依赖的最新版本，可以使用latest.integration。也可以使用加好('+')来指定版本。比如‘1.+’，表示1.x版本中最新的发布。

###文件依赖
当有些依赖在项目中时，比如在libs/cargo目录里，那么可以使用文件依赖加载。

```
拷贝之前下载的依赖
task copyDependenciesToLocalDir(type: Copy) {
   from configurations.cargo.asFileTree
   into "${System.properties['user.home']}/libs/cargo"
}
```
导入依赖

```
dependencies {
    cargo fileTree(dir: '/libs/cargo', include: '*.jar')
    //或者这样
    compile fileTree(include: '*.jar', dir: 'libs')
}
```

###使用并配置仓库
定义仓库的核心是RepositoryHandler接口，它提供方法来增加多种类型的仓库，maven，ivy和本地仓库。项目里通过repositories配置块来调用这些方法。当依赖管理器试图下载依赖和它的元数据是，它会以声明顺序检查仓库。第一个提供依赖的仓库成功后，接下来的仓库不会检查指定的依赖。

![](https://github.com/lihenair/Read-note/blob/master/image/repositoryhandler_uml.png)

####Maven仓库
Maven仓库是Java工程最常用的仓库。库文件通常是jar文件。元数据在POM文件中，以xml格式描述了库的信息和它的传递依赖。
引入Maven远程仓库：

```
repositories {
   mavenCentral()
}
```
依赖从仓库下载后将放到<USER_HOME>/.m2/repository目录中。

####增加自定义maven仓库

```
repositories {
   mavenCentral()
   maven {
      name 'Custom Maven Repository',
      url 'http://repository-gradle-in-action.forge.cloudbees.com/release/')
   }
}
```

####目录仓库
最简单最基本的仓库形式是目录仓库。该目录只包含jar文件，没有元数据。当声明依赖时，只需要使用name和version属性。group属性不会评估并会导致无法解决的依赖问题。使用方式如下：

```
repositories {
   flatDir(dir: "${System.properties['user.home']}/libs/cargo", name: 'Local libs directory')
}

dependencies {
   //完整模式
   cargo name: 'activation', version: '1.1'
   cargo name: 'ant', version: '1.7.1'
   cargo name: 'ant-launcher', version: '1.7.1'
   //简写模式
   cargo ':jaxb-api:2.1', ':jaxb-impl:2.1.13'
}
```

###理解本地依赖缓存
Gradle会自动决定一个是否需要从仓库下载并存到本地缓存。任何后续构建都讲师团重用这些下载好的依赖包。

####分析缓存结构

```
task printDependencies << {
   configurations.getByName('cargo').each { dependency ->
      println dependency
   }
}
```
该任务将打印工程所需要的所有缓存的依赖包。

Gradle保存依赖的本地缓存根目录是<USER_HOME>/.gradle/caches中，根据不同的gradle版本，内部的目录结构可能不同。

###排除依赖问题
可以指定依赖冲突策略实现构建失败

```
configurations.cargo.resolutionStrategy {
   failOnVersionConflict()
}
```

强制特定版本

```
configurations.cargo.resolutionStrategy {
   force 'org.codehaus.cargo-ant:1.3.0'
}
```

##Chapter 6 多模块构建
