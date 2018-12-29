---
layout:     post
title:      “invalid target release”错误fix
subtitle:   
date:       2018-12-29
author:     huanzhewu
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - 问题定位
---

从github上拉取了一个项目，在本地运行的时候，提示的错误：“error:java:invalid target release 8”

为了解决这个问题，可以打开 .idea/compiler.xml ,把配置文件中：

```java
<?xml version="1.0" encoding="UTF-8"?>
<project version="4">
  <component name="CompilerConfiguration">
    <annotationProcessing>
      <profile name="Maven default annotation processors profile" enabled="true">
        <sourceOutputDir name="target/generated-sources/annotations" />
        <sourceTestOutputDir name="target/generated-test-sources/test-annotations" />
        <outputRelativeToContentRoot value="true" />
        <module name="jnu_forum" />
      </profile>
    </annotationProcessing>
    <bytecodeTargetLevel target="7">
      <module name="jnu_forum" target="7" />
    </bytecodeTargetLevel>
  </component>
</project>
```

中的`<bytecodeTargetLevel target="7">`改成你的机器上对应的jdk版本。如果还不行，那你还需要确认下多处的jdk配置是否正常，请看下面这个文章：

https://stackoverflow.com/questions/25878045/errorjava-invalid-source-release-8-in-intellij-what-does-it-mean






