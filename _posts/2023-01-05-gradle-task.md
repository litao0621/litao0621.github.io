---
title: 'Gradle 学习总结 - Task-简单任务'
author: litao
date: 2023-01-05 19:13:00 +0800
categories: [Gradle]
tags: [Gradle,Android]


---

> 以下所有示例均基于Kotlin DSL

## 一、任务（Task）的定义

task通常分为简单任务与增强任务，此次只总结简单任务

- 简单任务：通常使用闭包在构建脚本中直接定义，适用于单次使用，无法提供给外部使用。
- 增强任务：通常是一个独立文件或独立项目，一般是打包为通用工具提供给外部使用，例如我们常用的`assembleRelease`,`clean`等

### 1. 创建任务

以下第二个方法中执行了Copy任务，通常还包含`Delete` `Copy` `Exec` `Zip`等常用任务。

```kotlin
//自定义任务
tasks.register("hello") {
    doLast {
        println("hello")
    }
}

//自定义任务及行为，可以指定系统包装好的一些通用行为
tasks.register<Copy>("copy") {
    from(file("srcDir"))
    into(buildDir)
}
```

当使用Kotlin DSL时还可以使用委托的方式来定义任务

```kotlin
val hello by tasks.registering {
    doLast {
        println("hello")
    }
}

val copy by tasks.registering(Copy::class) {
    from(file("srcDir"))
    into(buildDir)
}
```

### 2. 定位任务

通常如果希望将任务用作依赖项或进行配置时，可以通过`named`函数来获取到任务，进一步获取到该任务更详细的信息

```kotlin
tasks.register("hello")
tasks.register<Copy>("copy")

println(tasks.named("hello").get().name) 
//如果通过plugin方式添加的，还可以进一步简化
println(tasks.hello.get().name) 

println(tasks.named<Copy>("copy").get().destinationDir)
```

也可以通过`withType`函数来获取一组指定类型的任务

```kotlin
tasks.withType<Tar>().configureEach {
    enabled = false
}

tasks.register("test") {
    dependsOn(tasks.withType<Copy>())
}
```

也可以同过`getByPath`来获取任务，为了复杂任务的可维护性，最好是减少此方式的使用。

```kotlin
tasks.register("hello")

println(tasks.getByPath("hello").path)
println(tasks.getByPath(":hello").path)
println(tasks.getByPath("project-a:hello").path)
println(tasks.getByPath(":project-a:hello").path)
```

### 3. 任务的配置

列用copy类型任务通过内部API配置

```kotlin
tasks.named<Copy>("myCopy") {
    from("resources")
    into("target")
    include("**/*.txt", "**/*.xml", "**/*.properties")
}

//也可以将任务存储到变量中，后续进一步配置
val myCopy by tasks.existing(Copy::class) {
    from("resources")
    into("target")
}
myCopy {
    include("**/*.txt", "**/*.xml", "**/*.properties")
}

//也可以在任务定义时直接使用配置块
tasks.register<Copy>("copy") {
   from("resources")
   into("target")
   include("**/*.txt", "**/*.xml", "**/*.properties")
}
```

### 4. 将参数传递给任务的构造函数

定义任务时需要使用`@javax.inject.Inject`注解构造函数

```kotlin
abstract class CustomTask @Inject constructor(
    private val message: String,
    private val number: Int
) : DefaultTask()

//使用
tasks.register<CustomTask>("myTask", "hello", 42)
```

### 5. 添加任务依赖项

同项目直接使用名称进行依赖，不同的项目需要加项目名称的前缀

```kotlin
// path: project-a/build.gradle.kts
tasks.register("taskX") {
    dependsOn(":project-b:taskY")
    doLast {
        println("taskX")
    }
}

//path: project-b/build.gradle.kts
tasks.register("taskY") {
    doLast {
        println("taskY")
    }
}
```

同过task provider的方式进行依赖

```kotlin
val taskX by tasks.registering {
    doLast {
        println("taskX")
    }
}

val taskY by tasks.registering {
    doLast {
        println("taskY")
    }
}

taskX {
    dependsOn(taskY)
}
```

通过lazy block的方式进行依赖

```kotlin
val taskX by tasks.registering {
    doLast {
        println("taskX")
    }
}

// Using a Gradle Provider
taskX {
    dependsOn(provider {
        tasks.filter { task -> task.name.startsWith("lib") }
    })
}

tasks.register("lib1") {
    doLast {
        println("lib1")
    }
}

tasks.register("lib2") {
    doLast {
        println("lib2")
    }
}

tasks.register("notALib") {
    doLast {
        println("notALib")
    }
}
```

### 6. 任务顺序

通常我们可能在某些业务下需要保证任务的执行顺序，任务排序中有两种规则

- 必须在..之后允许（*must run after*）
- 应该在..之后允许（*should run after*）

感觉两种差异不是太大，在循环依赖时should run after将不再生效

```kotlin
val taskX by tasks.registering {
    doLast {
        println("taskX")
    }
}
val taskY by tasks.registering {
    doLast {
        println("taskY")
    }
}
val taskZ by tasks.registering {
    doLast {
        println("taskZ")
    }
}
taskX { dependsOn(taskY) }
taskY { dependsOn(taskZ) }
taskZ { shouldRunAfter(taskX) }

//执行 - gradle -q taskX
//taskZ
//taskY
//taskX
```



注意mustRunAfter与shouldRunAfter并不存在依赖关系，两个任务都是可以单独执行的，只有多任务都计划执行时排序才生效。

```kotlin
val taskX by tasks.registering {
    doLast {
        println("taskX")
    }
}
val taskY by tasks.registering {
    doLast {
        println("taskY")
    }
}


taskY {
    mustRunAfter(taskX)
}

taskY {
    shouldRunAfter(taskX)
}

//两种方式输出结果时一致的
//执行 - gradle -q taskY taskX
//taskX
//taskY
```

### 7. 任务中添加描述

通常我们会在任务中添加描述，在执行任务的时候会显示，便于试及使用者查看

```kotlin
tasks.register<Copy>("copy") {
   description = "Copies the resource directory to the target directory."
   from("resources")
   into("target")
   include("**/*.txt", "**/*.xml", "**/*.properties")
}
```

### 8. 任务的跳过

使用[Task.onlyIf](https://docs.gradle.org/current/dsl/org.gradle.api.Task.html#org.gradle.api.Task:onlyIf(org.gradle.api.specs.Spec)) 满足指定条件时执行，不满足则跳过，并且可以添加原因描述,使用 `--info`后缀来输出描述

```kotlin
val hello by tasks.registering {
    doLast {
        println("hello world")
    }
}

hello {
    val skipProvider = providers.gradleProperty("skipHello")
    onlyIf("there is no property skipHello") {
        !skipProvider.isPresent()
    }
}
//gradle hello -PskipHello --hello
```

某些情况可能无法使用`onlyif`谓词，可以使用[StopExecutionException](https://docs.gradle.org/current/userguide/more_about_tasks.html#sec:using_stopexecutionexception)，触发此异常时会跳过后续操作，继续构建下一个任务

```kotlin
val compile by tasks.registering {
    doLast {
        println("We are doing the compile.")
    }
}

compile {
    doFirst {
        // Here you would put arbitrary conditions in real life.
        if (true) {
            throw StopExecutionException()
        }
    }
}
tasks.register("myTask") {
    dependsOn(compile)
    doLast {
        println("I am not affected")
    }
}
```

通过指定任务的可用性来控制执行，使用`enabled`属性

```kotlin
val disableMe by tasks.registering {
    doLast {
        println("This should not be printed if the task is disabled.")
    }
}

disableMe {
    enabled = false
}
```

通过指定超时来限制执行时间

```kotlin
tasks.register("hangingTask") {
    doLast {
        Thread.sleep(100000)
    }
    timeout = Duration.ofMillis(500)
}
```

### 9. 任务添加规则

批量为任务添加规则

```kotlin
tasks.addRule("Pattern: ping<ID>") {
    val taskName = this
    if (startsWith("ping")) {
        task(taskName) {
            doLast {
                println("Pinging: " + (taskName.replace("ping", "")))
            }
        }
    }
}
```

在任务依赖上也同样会执行，即使没有此任务

```kotlin
tasks.addRule("Pattern: ping<ID>") {
    val taskName = this
    if (startsWith("ping")) {
        task(taskName) {
            doLast {
                println("Pinging: " + (taskName.replace("ping", "")))
            }
        }
    }
}

tasks.register("groupPing") {
    dependsOn("pingServer1", "pingServer2")
}

// 没有pingServer1 pingServer2任务，但添加的规则仍然会执行
//> gradle -q groupPing
//Pinging: Server1
//Pinging: Server2
```

### 10. 终结任务

添加一个终结任务，使用`finalizedBy`函数，使用场景 例如：我们一系列构建任务后无论成功失败都需要进行缓存清理时

```kotlin
val taskX by tasks.registering {
    doLast {
        println("taskX")
      	//终结任务无论失败与否都会被执行，即使当前任务抛出异常
      	//throw RuntimeException()
    }
}
val taskY by tasks.registering {
    doLast {
        println("taskY")
    }
}

taskX { finalizedBy(taskY) }
```



