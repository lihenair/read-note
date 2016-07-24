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

```compileJava
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
task printVersion(group: 'versioning', description: 'Prints project version.') << {   logger.quiet "Version: $version"}
```


task rule

```
task incrementMajorVersion(group: 'versioning', description: 'Increments project major version.') << {    String currentVersion = version.toString()    ++version.major    String newVersion = version.toString()    logger.info "Incrementing major project version: $currentVersion -> $newVersion"    ant.propertyfile(file: versionFile) {        entry(key: 'major', type: 'int', operation: '+', value: 1)    }
}task incrementMinorVersion(group: 'versioning', description: 'Increments project minor version.') << {    String currentVersion = version.toString()    ++version.minor    String newVersion = version.toString()
    logger.info "Incrementing minor project version: $currentVersion -> $newVersion"    ant.propertyfile(file: versionFile) {		entry(key: 'minor', type: 'int', operation: '+', value: 1)    }
}					
```