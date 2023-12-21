---
title: 'Gradle 学习总结 - Gradle插件发布'
author: litao
date: 2023-03-04 19:36:00 +0800
categories: [Gradle]
tags: [Gradle,Android]

---

## 一、Publish Plugin To Maven

插件的发布与Android 自定义库的发布相同，主要分为以下几个步骤

### 1. 注册Maven Central账号

流程网络上很多，不细说了，大致流程就是

- [Maven Central](https://issues.sonatype.org/)上注册一个账号
- Jira 上提交一个问题工单，简单来说就是帮你开辟一块属于你的空间，填写组织名称，项目名称，项目地址等一些信息
- 后续可能对方要验证一下你填写的合法性，例如使用GitHub作为项目仓库，可以会要求你创建一个指定项目进行验证。个人域名的话可能需要在DNS配置一条TXT记录来验证
- 提交的工单被标记为已解决后，表示我们已经有了属于自己的空间

### 2. 下面是配置发布脚本

以下是一个通用脚本，标记 ！！！的是你可能需要修改的地方

```groovy
apply plugin: 'com.android.library'
apply plugin: 'maven-publish'
apply plugin: 'signing'

task androidSourcesJar(type: Jar) {
    archiveClassifier.set("sources")
    from android.sourceSets.main.java.source
    exclude "**/R.class"
    exclude "**/BuildConfig.class"
}


publishing {
    publications {
        release(MavenPublication) {
            def VERSION_NAME = PUBLISH_VERSION_NAME as String
            def GROUP_ID = PUBLISH_GROUP_ID as String

            println("maven-publish:  publish module (${project.getName()}) -> $GROUP_ID : $PUBLISH_ARTIFACT_ID - $VERSION_NAME")

            groupId GROUP_ID
            artifactId PUBLISH_ARTIFACT_ID
            version VERSION_NAME

          	//以下按需配置
            artifact("$buildDir/outputs/aar/${project.getName()}-release.aar")
            artifact androidSourcesJar

            pom {
                name = PUBLISH_ARTIFACT_ID
                description = 'test plugin.' //!!!
                url = 'https://github.com/litao0621/MyPlugin' //!!!
                licenses {
                    license {
                        name = 'The Apache License, Version 2.0'
                        url = 'http://www.apache.org/licenses/LICENSE-2.0.txt'
                    }
                }
                developers {
                    developer {
                        id = 'litao' //!!!
                        name = 'li tao' //!!!
                        email = 'onresume@live.com' //!!!
                    }
                }
                scm {
                    connection = 'scm:git@github.com:litao0621/MyPlugin.git' //!!!
                    developerConnection = 'git@github.com:litao0621/MyPlugin.git' //!!!
                    url = 'https://github.com/litao0621/MyPlugin/tree/main' //!!!
                }
                withXml {
                    def dependenciesNode = asNode().appendNode('dependencies')

                    project.configurations.implementation.allDependencies.each {
                        def dependencyNode = dependenciesNode.appendNode('dependency')
                        dependencyNode.appendNode('groupId', it.group)
                        dependencyNode.appendNode('artifactId', it.name)
                        dependencyNode.appendNode('version', it.version)
                    }
                }
            }
        }
    }
    repositories {
        maven {
            name = "MyPlugin" //!!!
            def releasesRepoUrl = "https://s01.oss.sonatype.org/service/local/staging/deploy/maven2/"
            def snapshotsRepoUrl = "https://s01.oss.sonatype.org/content/repositories/snapshots/"
            url = version.endsWith('SNAPSHOT') ? snapshotsRepoUrl : releasesRepoUrl

            credentials {
                username ossrhUsername
                password ossrhPassword
            }
        }
    }
}

signing {
    sign publishing.publications
}
```

注意几点

- 隐私性叫高的尽量配置在个人设备系统目录下的`.gradle`文件夹下的 `gradle.properties`中避免外泄。例如账号、密码、证书信息等。
- 不涉及隐私的但需要动态配置的可以直接放在项目下或的`gradle.properties文件`，如项目名称,版本等

### 3. 创建密钥

发布还需要一个密钥，流程网络上也很多，不展开细说，简单介绍下流程

- 先安装一个密钥管理得客户端，这里推荐[GPGTools](https://gpgtools.org/) 
- 创建一个密钥
- 发布密钥到服务器

### 4. 打包待发布资源

执行`maven-publish`插件中的`publishXXX`即可，等待成功看到发布成功的提示

### 5. 发布项目

登录我们脚本中填写的 仓库管理地址[`https://s01.oss.sonatype.org`](https://s01.oss.sonatype.org) 找到对应项目 点击 `close` 任务完成后再点击 `release` 即可。

### 6. 插件引用

到这里就可以引用我们的插件了

```kotlin
// project/build.gradle
buildscript {
    repositories {
        google()
        mavenCentral()

    }
    dependencies {
        classpath("io.github.litao0621:MyPlugin:1.0.0")
    }
}
```

```kotlin
//app/build.gradle
plugins {
    id("test-plugin")
}
```

