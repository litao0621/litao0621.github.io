---
title: 'Gradle 学习总结 - Task-增强任务'
author: litao
date: 2023-01-06 11:06:00 +0800
categories: [Gradle]
tags: [Gradle,Android]



---

> 以下所有示例均基于Kotlin DSL

## 一、增强任务

增强任务通常被定义为一个独立的模块，例如我们通常使用的Gradle插件都是使用的增强任务。主要优势是便于更灵活的配置，能够在不同项目不同地方提供行为的重用。

### 1. 任务类的封装

可以通过以下三种方式

- Gradle 脚本文件中直接定义，无需额外配置，但仅脚本文件内可用。
- 在`buildSrc`模块中，kotlin源代码位置 - `rootProjectDir/buildSrc/src/main/kotlin` ,`buildSrc属于gradle项目中的一个特殊目录，会自动编译，无需额外配置，再当前gradle项目是可见的。
- 编写独立项目，可以生成一个独立的jar，提供给任何位置使用，需要进行简单配置。这种也是最长用的。

### 2. 简单任务类的定义

以下时一个简单的任务类，如以下片段 使用 [@TaskAction](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/TaskAction.html) 进行注解，任务执行时将自动调用此方法，以下还添加了一个配置参数，供调用者配置。

```kotlin
abstract class GreetingTask : DefaultTask() {
    @get:Input
    abstract val greeting: Property<String>

    init {
        greeting.convention("hello from GreetingTask")
    }

    @TaskAction
    fun greet() {
        println(greeting.get())
    }
}

// Use the default greeting
tasks.register<GreetingTask>("hello")

// Customize the greeting
tasks.register<GreetingTask>("greeting") {
    greeting = "greetings from GreetingTask"
}
```

### 3. 独立项目的任务

以下使用groovy插件 并将Gradle API添加为编译时依赖项

```kotlin
// build.gradle.kts
plugins {
    groovy
}

dependencies {
    implementation(gradleApi())
}
```

独立类中编写自定义任务

```kotlin
//src/main/groovy/org/gradle/GreetingTask.groovy
package org.gradle

import org.gradle.api.DefaultTask
import org.gradle.api.tasks.TaskAction
import org.gradle.api.tasks.Input

class GreetingTask extends DefaultTask {

    @Input
    String greeting = 'hello from GreetingTask'

    @TaskAction
    def greet() {
        println greeting
    }
}
```

在独立项目中进行依赖，引用已经发布到本地的JAR

```kotlin
buildscript {
    repositories {
        maven {
            url = uri(repoLocation)
        }
    }
    dependencies {
        classpath("org.gradle:task:1.0-SNAPSHOT")
    }
}

tasks.register<org.gradle.GreetingTask>("greeting") {
    greeting = "howdy!"
}
```

### 3. 增量任务

如果您想优化构建以便仅处理过时的输入文件，您可以通过*增量任务*，对于增量处理输入的任务，该任务必须包含*增量任务操作*。 这是一个任务操作方法，具有 [InputChanges](https://docs.gradle.org/current/dsl/org.gradle.work.InputChanges.html) 参数。 该参数告诉 Gradle 该操作只想处理更改的输入。此外，任务需要使用 [@Incremental](https://docs.gradle.org/current/javadoc/org/gradle/work/Incremental.html) 或 [@SkipWhenEmpty](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/SkipWhenEmpty.html) 增量文件输入属性

要查询输入文件属性的增量更改，该属性始终需要返回相同的实例。实现这一点的最简单方法是对此类属性使用以下类型之一：[RegularFileProperty](https://docs.gradle.org/current/javadoc/org/gradle/api/file/RegularFileProperty.html), [DirectoryProperty](https://docs.gradle.org/current/javadoc/org/gradle/api/file/DirectoryProperty.html) or [ConfigurableFileCollection](https://docs.gradle.org/current/javadoc/org/gradle/api/file/ConfigurableFileCollection.html).

```kotlin
abstract class IncrementalReverseTask : DefaultTask() {
    @get:Incremental
    @get:PathSensitive(PathSensitivity.NAME_ONLY)
    @get:InputDirectory
    abstract val inputDir: DirectoryProperty

    @get:OutputDirectory
    abstract val outputDir: DirectoryProperty

    @get:Input
    abstract val inputProperty: Property<String>

    @TaskAction
    fun execute(inputChanges: InputChanges) {
        println(
            if (inputChanges.isIncremental) "Executing incrementally"
            else "Executing non-incrementally"
        )

        inputChanges.getFileChanges(inputDir).forEach { change ->
            if (change.fileType == FileType.DIRECTORY) return@forEach

            println("${change.changeType}: ${change.normalizedPath}")
            val targetFile = outputDir.file(change.normalizedPath).get().asFile
            if (change.changeType == ChangeType.REMOVED) {
                targetFile.delete()
            } else {
                targetFile.writeText(change.file.readText().reversed())
            }
        }
    }
}
```

### 4. 通过命令行配置参数

只需要通过[@Option](https://docs.gradle.org/current/javadoc/org/gradle/api/tasks/options/Option.html) 进行注解相应的`setter`方法

```kotlin
import org.gradle.api.tasks.options.Option;

public class UrlVerify extends DefaultTask {
    private String url;

    @Option(option = "url", description = "Configures the URL to be verified.")
    public void setUrl(String url) {
        this.url = url;
    }

    @Input
    public String getUrl() {
        return url;
    }

    @TaskAction
    public void verify() {
        getLogger().quiet("Verifying URL '{}'", url);

        // verify URL by making a HTTP call
    }
}
```

使用时在命令中使用双破折号添加参数

```kotlin
tasks.register<UrlVerify>("verifyUrl")
```



```bash
gradle -q verifyUrl --url=http://www.google.com/
```

以下是命令行所支持的参数类型

	- `boolean`**,** `Boolean`**,** `Property<Boolean>`
	- `Double`**,** `Property<Double>`
	- `Integer`**,** `Property<Integer>`
	- `Long`, `Property<Long>`
	- `String`**,** `Property<String>`
	- `enum`**,** `Property<enum>`
	- `List<T>` **where** `T` **is** `Double`**,** `Integer`**,** `Long`**,** `String`**,** `enum`
	- `ListProperty<T>`**,** `SetProperty<T>` **where** `T` **is** `Double`**,** `Integer`**,** `Long`**,** `String`**,** `enum`
	- `DirectoryProperty`**,** `RegularFileProperty`

