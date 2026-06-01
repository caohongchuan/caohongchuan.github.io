---
title: SpringBoot 多项目工程（基于Gradle）
date: 2026-05-30 00:00:00 +0800
categories: [spring, springcloud]
tags: [spring]
author: caohongchuan
pin: false
math: true
toc: true
comments: true
---

## Gradle利用basic模板创建空父项目

```bash
mkdir spring-cloud-project
cd spring-cloud-project
# 初始化 gradle 根项目（选择 basic 类型）
gradle init --type basic --dsl groovy
```

初始化后根目录的结构

```
.
├── build.gradle       # 空
├── gradle
│   ├── ...
├── gradle.properties
├── gradlew
├── gradlew.bat
└── settings.gradle    # 只有项目名
```

其中核心为`build.gradle`和`settings.gradle`两个配置文件。

除了上述用命令行创建，也可通过IDEA先创建gradle为java-application的项目，然后手动删除src目录和修改`build.gradle`文件即可。

```
File -> New -> Project -> java -> build system(gradle)
```

<img src="https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20260530193101256.png" alt="image-20260530193101256" style="zoom:50%;" />

然后删除根项目中的`/src`文件夹和清空`build.gradle`的内容。（最终效果与命令行创建basic模板的gradle是一致的）

## 创建SpringBoot子项目

IDEA在根项目中创建Module

```
File -> New -> Module -> Generators(Spring Boot)
```

<img src="https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20260530194400173.png" alt="image-20260530194400173" style="zoom:50%;" />

依赖只添加Web用于测试

<img src="https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20260530194502028.png" alt="image-20260530194502028" style="zoom:50%;" />

新创建的子项目service1的目录，并删除其中的部分文件：

```
# service1 目录
.
├── build.gradle
├── gradle            删除
│   └── wrapper
├── gradlew           删除
├── gradlew.bat       删除
├── HELP.md
├── service1.iml
├── settings.gradle   删除（子模块不需要）
└── src
    ├── main
    └── test
```

注：只保留根目录的 `gradlew` 和 `gradle/`，统一管理Gradle。

在根项目的`settings.gradle`中添加子项目，纳入管理。

```groovy
// settings.gradle
rootProject.name = 'spring-cloud-project'

include ':service1'
```

注：若IDEA的gradle工具刷新后（建议关闭IDEA重新加载项目）同时出现了根项目（spring-cloud-project）和子项目（service1），或出现了多个根项目（spring-cloud-project），则只保留一个根项目（spring-cloud-project），将其他的项目都删掉（unlink gralde project）。

<img src="https://raw.githubusercontent.com/caohongchuan/blogimg/main/nextimg/image-20260530195836667.png" alt="image-20260530195836667" style="zoom: 67%;" />

其他子项目也通过相同的方式创建。

## 配置 build.gralde 文件

```
my-springboot-multi-project/
├── build.gradle          // 父项目构建脚本
├── settings.gradle       // 声明包含哪些子项目
├── gradle.properties     // （可选）定义公共属性
├── module-service1/             // 子项目1
│   └── build.gradle
└── module-service2/             // 子项目2
    └── build.gradle
```

### **父项目** `build.gradle` 

父项目的 `build.gradle` 主要用于（**统一版本管理、统一仓库源、统一编译环境**）：

- 定义公共插件（如 Java、Spring Boot）
- 配置仓库（repositories）
- 统一依赖版本（通过 `ext` 或 `dependencyManagement`）
- 应用通用配置到所有子项目（使用 `subprojects`）

```groovy
plugins {
    id 'org.springframework.boot' version '4.0.6' apply false
}

ext {
    set('springCloudVersion', '2025.1.0')
    set('springCloudAlibabaVersion', '2025.1.0.0')
}

allprojects {
    group = 'top.chc'
    version = '1.0.0'

    repositories {
        // 私有 Nexus（按需开启）
        // maven {
        //     name = '私有仓库'
        //     url  = 'http://nexus.example.com/repository/maven-public/'
        //     credentials {
        //         username = project.findProperty('nexusUser')     ?: ''
        //         password = project.findProperty('nexusPassword') ?: ''
        //     }
        //     allowInsecureProtocol = true   // 仅 http 时需要
        // }
        mavenCentral()
    }

}

subprojects {

    plugins.withType(JavaPlugin) {
        java {
            toolchain {
                languageVersion = JavaLanguageVersion.of(25)
            }
        }

        tasks.withType(JavaCompile).configureEach {
            options.encoding = 'UTF-8'
        }

        dependencies {
            implementation platform(
                    org.springframework.boot.gradle.plugin.SpringBootPlugin.BOM_COORDINATES
            )
            implementation enforcedPlatform(
                    "org.springframework.cloud:spring-cloud-dependencies:${springCloudVersion}"
            )
            implementation enforcedPlatform(
                    "com.alibaba.cloud:spring-cloud-alibaba-dependencies:${springCloudAlibabaVersion}"
            )
        }
    }

    plugins.withType(org.springframework.boot.gradle.plugin.SpringBootPlugin) {
        bootJar {
            enabled = true
        }
        jar {
            enabled = false
        }
    }
}
```

* `plugins`指定springboot的项目版本
* `ext`指定springcloud版本，springcloudalibaba版本
* `allprojects`指定项目名称版本，maven仓库地址
* `subprojects` 如果是Java项目指定Java版本；项目编码；springboot,springcloud,springcloudalibaba的BOM版本控制。如果是Springboot项目开启bootJar（编译成bootJar）关闭jar（不再编译成Jar）。

### 子项目 build.gradle

子项目只需要写需要的组件，不需要填写具体的版本号。

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot'
}

description = 'service1'

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-webmvc'
    testImplementation 'org.springframework.boot:spring-boot-starter-webmvc-test'
    testRuntimeOnly 'org.junit.platform:junit-platform-launcher'
}

tasks.named('test') {
    useJUnitPlatform()
}
```

