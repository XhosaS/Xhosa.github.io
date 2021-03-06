---
layout: post
title: 如何将JavaDoc转换为.rtf等格式
category: Android
tags: Andorid
description: 如何将JavaDoc转换为.rtf等格式
---

1. 首先去 docflex-doclet 官网下载一个最新版的包，把里面的 docflex-doclet.jar 拷贝到一个方便你用的地方。我是放在`D:\Project\ODM_APP\XgimiOpenAPI\docflex-doclet.jar`里的。

2. 在 Android Studio 上用 `Tools` -> `Generate javaDoc`

3. 选择需要生成的文档，参数里面填

Other command line arguments: `-doclet com.docflex.javadoc.Doclet -docletpath D:\Project\ODM_APP\XgimiOpenAPI\docflex-doclet.jar -encoding utf-8 -charset utf-8`

4. 选择 `OK`



5. 这时，在 Run 窗口的第一行，会看到 Android Studio 的 Generate javaDoc 工具帮你生成的`javadoc`命令。

不出意外的话，会编译报错:`无效的标记: -Xdoclint:none`。那是因为 Doclet 1.6.1只支持 `Java7`，而这个标记是`Java8`专有的。



6. 于是，我们把这行命令的`-Xdoclint:none`删掉，然后放到 Terminal 中执行（自己注意改一下里面的路径）：

```powershell
"C:\Program Files\Android\Android Studio\jre\bin\javadoc.exe" -protected -splitindex -use -author -version -doclet com.docflex.javadoc.Doclet -docletpath D:\Project\ODM_APP\XgimiOpenAPI\docflex-doclet.jar -encoding utf-8 -charset utf-8 -d D:\Project\XgimiOpenAPI\XgimiOpenAPI2.0\2.0 @C:\Users\xhosa\AppData\Local\Temp\javadoc_args -bootclasspath C:\Users\xhosa\AppData\Local\Android\Sdk\platforms\android-27\android.jar

```

这时候应该又会报错，无法读取 `C:\Users\xhosa\AppData\Local\Temp\javadoc_args`。



7. 然后我们回到Run窗口，把`@C:\Users\xhosa\AppData\Local\Temp\javadoc_args`中的文本复制出来，然后再目标目录创建一个该文件，并把文本复制进去，这时候再执行，应该就能成功了。


8. 这时候，docflex-doclet.jar 就会弹出来，然后配置一下，生成`.rtf`文件。


9. 用 Office word 打开`.rtf`文件，就可以导出为各种格式了。
