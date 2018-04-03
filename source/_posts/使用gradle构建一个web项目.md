---
title: 使用gradle构建一个web项目
date: 2016-05-19 21:49:03
tags: gradle
categories: 构建工具 
---

安装过程略去，配置下GRADLE_HOME和GRADLE_HOME\bin

1. 创建一个空目录，新建build.gradle

```groovy
apply plugin: 'idea'
apply plugin: 'java' 
apply plugin: 'war'
sourceCompatibility = 1.7

repositories {
  mavenCentral()
}

dependencies {
  compile 'org.springframework.boot:spring-boot-starter-web:1.3.5.RELEASE'
  compile 'log4j:log4j:1.2.17'
}


task createJavaProject << { 
  sourceSets*.java.srcDirs*.each { it.mkdirs() } 
  sourceSets*.resources.srcDirs*.each { it.mkdirs()} 
} 

task createWebProject(dependsOn: 'createJavaProject') << { 
  def webAppDir = file("$webAppDirName") 
  webAppDir.mkdirs() 
} 
```

2.gradle idea
3.gradle createWebProject
3.gradle build