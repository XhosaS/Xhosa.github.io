---
layout: post
title: 如何在Android Framework中加入一个JNI实现的API
category: Android
tags: Andorid
description: 如何在Android Framework中加入一个JNI实现的API
---

# 添加JNI接口

在`/frameworks/base/core/jni`路径下创建`android_util_MyEncryption.h`文件和`android_util_MyEncryption.cpp`文件,这个文件是 JNI 层的实现接口。

```c++
#ifndef _ANDROID_UTIL_MYENCRYPTION_H
#define _ANDROID_UTIL_MYENCRYPTION_H

#include <jni.h>
#include <JNIHelp.h>

namespace android {

}

#endif //_ANDROID_UTIL_MYENCRYPTION_H
```
`android_util_MyEncryption.cpp`文件：
```c++
#include "jni.h"
#include "JNIHelp.h"
#include "core_jni_helpers.h"
#include "android_util_MyEncryption.h"

namespace android {

static jstring android_util_MyEncryption_getHashValue(JNIEnv *env, jclass clazz,jstring text) {
    return env->NewStringUTF("test");
}

/* 
* JNI registration. 
*/  
static JNINativeMethod gMethods[] = {  
    /* name, signature, funcPtr */  
    {"getHashValue","(Ljava/lang/String;)Ljava/lang/String;",  (void*) android_util_MyEncryption_getHashValue },  
      
};  

int register_android_util_MyEncryption(JNIEnv* env)
{
    return RegisterMethodsOrDie(env, "android/util/MyEncryption", gMethods, NELEM(gMethods));
}

}; // namespace android
```



# 增加编译的修改配置

修改`/frameworks/base/core/jni/Android.mk`文件

```makefile
@@ -87,6 +87,7 @@ LOCAL_SRC_FILES:= \
     android_util_Binder.cpp \
     android_util_EventLog.cpp \
     android_util_Log.cpp \
+    android_util_MyEncryption.cpp \
     android_util_Process.cpp \
     android_util_StringBlock.cpp \
     android_util_XmlBlock.cpp \
```



修改`/frameworks/base/core/jni/AndroidRuntime.cpp`文件

```cpp
@@ -108,6 +108,7 @@ namespace android {
 extern int register_android_content_AssetManager(JNIEnv* env);
 extern int register_android_util_EventLog(JNIEnv* env);
 extern int register_android_util_Log(JNIEnv* env);
+extern int register_android_util_MyEncryption(JNIEnv* env);
 extern int register_android_content_StringBlock(JNIEnv* env);
 extern int register_android_content_XmlBlock(JNIEnv* env);
 extern int register_android_emoji_EmojiFactory(JNIEnv* env);
@@ -1302,6 +1303,7 @@ static const RegJNIRec gRegJNI[] = {
     REG_JNI(register_android_os_SystemClock),
     REG_JNI(register_android_util_EventLog),
     REG_JNI(register_android_util_Log),
+    REG_JNI(register_android_util_MyEncryption),
     REG_JNI(register_android_content_AssetManager),
     REG_JNI(register_android_content_StringBlock),
     REG_JNI(register_android_content_XmlBlock),
```



# 添加JAVA文件

在`frameworks/base/core/java/android/util/`新建文件`MyEncryption.java`

```java
package android.util;


public final class MyEncryption {
    
    private MyEncryption() {

    }

    public static native String getHashValue(String text);
}
```



完成后，重新编译，就可以使用这个类了。



# 添加校验机制

虽然将实现手段下沉到了 JNI 层，但是非授权用户还是可以通过反射的方式调用这个函数，可以在 JNI 层添加签名和包名的校验来防止这种情况。在`android_util_MyEncryption.cpp`中添加校验。

```c++
#include "jni.h"
#include "JNIHelp.h"
#include "core_jni_helpers.h" 
#include "android_util_MyEncryption.h"

namespace android {

static jobject getApplication(JNIEnv *env) {
    jobject application = NULL;
    jclass activity_thread_clz = env->FindClass("android/app/ActivityThread");
    if (activity_thread_clz != NULL) {
        jmethodID currentApplication = env->GetStaticMethodID(
                activity_thread_clz, "currentApplication", "()Landroid/app/Application;");
        if (currentApplication != NULL) {
            application = env->CallStaticObjectMethod(activity_thread_clz, currentApplication);
        }
        env->DeleteLocalRef(activity_thread_clz);
    }
    return application;
}


/**
 * HexToString
 * @param source
 * @param dest
 * @param sourceLen
 */
static void ToHexStr(const char *source, char *dest, int sourceLen) {
    short i;
    char highByte, lowByte;

    for (i = 0; i < sourceLen; i++) {
        highByte = source[i] >> 4;
        lowByte = (char) (source[i] & 0x0f);
        highByte += 0x30;

        if (highByte > 0x39) {
            dest[i * 2] = (char) (highByte + 0x07);
        } else {
            dest[i * 2] = highByte;
        }

        lowByte += 0x30;
        if (lowByte > 0x39) {
            dest[i * 2 + 1] = (char) (lowByte + 0x07);
        } else {
            dest[i * 2 + 1] = lowByte;
        }
    }
}

/**
 *
 * byteArrayToMd5
 * @param env
 * @param source
 * @return j_string
 */
static jstring ToMd5(JNIEnv *env, jbyteArray source) {
    // MessageDigest
    jclass classMessageDigest = env->FindClass("java/security/MessageDigest");
    // MessageDigest.getInstance()
    jmethodID midGetInstance = env->GetStaticMethodID(classMessageDigest, "getInstance",
                                                      "(Ljava/lang/String;)Ljava/security/MessageDigest;");
    // MessageDigest object
    jobject objMessageDigest = env->CallStaticObjectMethod(classMessageDigest, midGetInstance,
                                                           env->NewStringUTF("md5"));

    jmethodID midUpdate = env->GetMethodID(classMessageDigest, "update", "([B)V");
    env->CallVoidMethod(objMessageDigest, midUpdate, source);

    // Digest
    jmethodID midDigest = env->GetMethodID(classMessageDigest, "digest", "()[B");
    jbyteArray objArraySign = (jbyteArray) env->CallObjectMethod(objMessageDigest, midDigest);

    jsize intArrayLength = env->GetArrayLength(objArraySign);
    jbyte *byte_array_elements = env->GetByteArrayElements(objArraySign, NULL);
    size_t length = (size_t) intArrayLength * 2 + 1;
    char *char_result = (char *) malloc(length);
    memset(char_result, 0, length);

    ToHexStr((const char *) byte_array_elements, char_result, intArrayLength);
    // 在末尾补\0
    *(char_result + intArrayLength * 2) = '\0';

    jstring stringResult = env->NewStringUTF(char_result);
    // release
    env->ReleaseByteArrayElements(objArraySign, byte_array_elements, JNI_ABORT);
    // 指针
    free(char_result);

    return stringResult;
}

static jstring android_util_MyEncryption_getHashValue(JNIEnv *env, jclass clazz,jstring text) {
    //校验签名和包名
    //获得Context类
    jobject context = getApplication(env);
    jclass cls = env->GetObjectClass(context);
    //得到getPackageManager方法的ID
    jmethodID mid = env->GetMethodID(cls, "getPackageManager","()Landroid/content/pm/PackageManager;");
    //获得应用包的管理器
    jobject pm = env->CallObjectMethod(context, mid);
    //得到getPackageName方法的ID
    mid = env->GetMethodID(cls, "getPackageName", "()Ljava/lang/String;");
    //获得当前应用包名
    jstring packageName = (jstring) env->CallObjectMethod(context, mid);
    const char *c_pack_name = env->GetStringUTFChars(packageName, NULL);
    if (strcmp(c_pack_name, "android") != 0) {
        return (env)->NewStringUTF(c_pack_name);
    }
    // 获得PackageManager类
    cls = env->GetObjectClass(pm);
    mid = env->GetMethodID(cls, "getPackageInfo",
                           "(Ljava/lang/String;I)Landroid/content/pm/PackageInfo;");
                           // 获得应用包的信息
    jobject packageInfo = env->CallObjectMethod(pm, mid, packageName, 0x40); //GET_SIGNATURES = 64;
    // 获得PackageInfo 类
    cls = env->GetObjectClass(packageInfo);
    // 获得签名数组属性的ID
    jfieldID fid = env->GetFieldID(cls, "signatures", "[Landroid/content/pm/Signature;");
    // 得到签名数组
    jobjectArray signatures = (jobjectArray) env->GetObjectField(packageInfo, fid);
    // 得到签名
    jobject signature = env->GetObjectArrayElement(signatures, 0);

    // 获得Signature类
    cls = env->GetObjectClass(signature);
    mid = env->GetMethodID(cls, "toByteArray", "()[B");
    // 当前应用签名信息
    jbyteArray signatureByteArray = (jbyteArray) env->CallObjectMethod(signature, mid);
    //转成jstring
    jstring str = ToMd5(env, signatureByteArray);
    char *c_msg = (char *) env->GetStringUTFChars(str, 0);
    if (strcmp(c_msg, "YOUR SIGNATURE") != 0) {
        return (env)->NewStringUTF(c_msg);
    }
    return env->NewStringUTF("test");
}
/* 
* JNI registration. 
*/  
static JNINativeMethod gMethods[] = {
    /* name, signature, funcPtr */  
    {"getHashValue",      "(Ljava/lang/String;)Ljava/lang/String;",  (void*) android_util_MyEncryption_getHashValue },
      
};  

int register_android_util_MyEncryption(JNIEnv* env)
{
    return RegisterMethodsOrDie(env, "android/util/MyEncryption", gMethods, NELEM(gMethods));
}

} // namespace android
```

# TIPS

`JNI FatalError called: RegisterNatives failed for`

编译完成后，开机过程中出现崩溃，拷出`core_dump`查看，发现`android_util_MyEncryption.cpp`中的`register_android_util_MyEncryption(JNIEnv* env)`会在开机时执行，来注册JNI库，一定要注意`gMethods`中的`signature `是否正确描述了 Java 中函数的参数和返回值 ，如果错误，会导致开机失败。
