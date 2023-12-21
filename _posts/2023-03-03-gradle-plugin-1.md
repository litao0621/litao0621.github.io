---
title: 'Gradle 学习总结 - Gradle插件开发'
author: litao
date: 2023-03-03 18:06:00 +0800
categories: [Gradle]
tags: [Gradle,Android]
---

插件的开发与任务类似，同样可以在构建脚本中、buildSrc模块、独立项目进行开发。通常我们都是会打包为一个独立项目。

## 一、插件的开发

### 1. 简单的插件

可以在构建脚本中直接定义插件，但通常很少这样用，可扩展性较低，感觉只适用于一些很简单的任务。以下是一个简单插件，并提供了参数的配置。参数配置是非必要的。

```kotlin

interface GreetingPluginExtension {
    val message: Property<String>
    val greeter: Property<String>
}

class GreetingPlugin : Plugin<Project> {
    override fun apply(project: Project) {
        val extension = project.extensions.create<GreetingPluginExtension>("greeting")
        project.task("hello") {
            doLast {
                println("${extension.message.get()} from ${extension.greeter.get()}")
            }
        }
    }
}

apply<GreetingPlugin>()

// Configure the extension using a DSL block
configure<GreetingPluginExtension> {
    message = "Hi"
    greeter = "Gradle"
}
```

插件中一半处理文件是一个很常用的功能，通常使用Gradle中的托管属性`project.layout来选择文件目录或位置，这种方式文件将被仅在需要时被解析。

```kotlin
abstract class GreetingToFileTask : DefaultTask() {

    @get:OutputFile
    abstract val destination: RegularFileProperty

    @TaskAction
    fun greet() {
        val file = destination.get().asFile
        file.parentFile.mkdirs()
        file.writeText("Hello!")
    }
}

val greetingFile = objects.fileProperty()

tasks.register<GreetingToFileTask>("greet") {
    destination = greetingFile
}

tasks.register("sayGreeting") {
    dependsOn("greet")
    val greetingFile = greetingFile
    doLast {
        val file = greetingFile.get().asFile
        println("${file.readText()} (file: ${file.name})")
    }
}

greetingFile = layout.buildDirectory.file("hello.txt")
```

### 2. 独立项目插件

通常我们使用 `java-gradle-plugin`插件来辅助gradle 插件的开发，为我们处理一些插件开发的依赖及打包，发布所需要的一些信息。接下来创建插件ID，插件ID的命名类似于android的包名，例如：`com.github.litao.plugin`，

```kotlin
//build.gradle.kts
plugins {
    `java-gradle-plugin`
}

gradlePlugin {
    plugins {
        create("simplePlugin") {
            id = "com.github.litao.plugin"
            implementationClass = "com.github.litao.plugin.MyPlugin"
        }
    }
}
```

定义我们的插件，以下同时展示了在一个插件中引用另一个插件

```kotlin
//MyBasePlugin.java
import org.gradle.api.Plugin;
import org.gradle.api.Project;

public class MyBasePlugin implements Plugin<Project> {
    public void apply(Project project) {
        // define capabilities
    }
}
```

```kotlin
//MyPlugin.java
import org.gradle.api.Plugin;
import org.gradle.api.Project;

public class MyPlugin implements Plugin<Project> {
    public void apply(Project project) {
        project.getPlugins().apply(MyBasePlugin.class);

        // define conventions
    }
}
```

### 3. 插件的发布

为了复用，我们通常会将插件发布到远程仓库，通常是`maven` 或 `gradle官方仓库`，这里我们选择`maven`，以下我们展示发布到本地maven 仓库，后面会单独说下远程仓库的发布及发布脚本的配置。发布到`maven`通常依赖 `maven-publish`将为我们减少大量工作。

以下是构建脚本中的一些配置

```kotlin
//settings.gradle.kts
pluginManagement {
    repositories {
        maven {
            url = uri(repoLocation)
        }
    }
}
```

```kotlin
//build.gradle.kts
plugins {
    id("org.example.greeting") version "1.0-SNAPSHOT"
}
```

注意：在没有使用 [`java-gradle-plugin`](https://docs.gradle.org/current/userguide/custom_plugins.html#note_for_plugins_published_without_java_gradle_plugin)插件发布时 将缺少  [Plugin Marker Artifact](https://docs.gradle.org/current/userguide/plugins.html#sec:plugin_markers) 无法直接引，可以通过在`resolutionStrategy`中添加`resolutionStrategy`来进行引用

```kotlin
//settings.gradle.kts
resolutionStrategy {
    eachPlugin {
        if (requested.id.namespace == "org.example") {
            useModule("org.example:custom-plugin:${requested.version}")
        }
    }
}
```

