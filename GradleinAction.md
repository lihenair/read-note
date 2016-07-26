#Gradle实践笔记#

##Chapter 3

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
##Chapter 4

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

###