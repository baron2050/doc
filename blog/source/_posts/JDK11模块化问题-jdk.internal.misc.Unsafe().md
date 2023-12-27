---
title: JDK11模块化问题-jdk.internal.misc.Unsafe()
---

JDK11使用netty等类库时有时会在DEBUG日志中出现以下错误:
java.lang.UnsupportedOperationException: Reflective setAccessible(true) disabled；
java.lang.IllegalAccessException: class io.netty.util.internal.PlatformDependent0$6 cannot access class jdk.internal.misc.Unsafe (in module java.base) because module java.base does not export jdk.internal.misc to unnamed module @60438a68；
虽然程序可以正常运行，但还是想解决这个问题，方案如下:
添加VM OPTIONS:
--add-opens java.base/jdk.internal.misc=ALL-UNNAMED
-Dio.netty.tryReflectionSetAccessible=true
--illegal-access=warn(可选)