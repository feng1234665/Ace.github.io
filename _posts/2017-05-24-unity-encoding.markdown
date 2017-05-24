---
layout: post
title: unity dll加密
date: 2017-05-24
---

## 1.1   加密方案

Unity 3D项目游戏逻辑采用C#脚本，我们知道C#编译生成的DLL或EXE是IL程序集。IL程序集中有一个MetaData，记录了程序集中的一切信息，所以容易被反编译。传统的防破解方式是是对IL程序集进行混淆或者加壳。但是这种混淆基本上只是做一些名称混淆或流程混淆或者加一些打花指令。这种混淆或加壳的结果基本上还是保留了IL程序集的原貌，还是很容易被破解的。

因为有这些缺点，我们实现了一套自己的加密方案：直接对IL程序集进行加密。改变程序集形态，这样子，它就不再是IL程序集格式了。

Unity 3D项目的游戏逻辑脚本编译后会生成Assembly-CSharp.dll。只要对它进行加密，但是这带来一个问题是，Unity不认加密后的DLL，因为它不再是dll格式了。那我们就需要修改Unity底层代码，好在Unity是基于Mono的，而Mono是开源的。只要找到Mono加载dll的代码，对它进行逆向即可。

Mono加载dll的代码位于/mono/metadata/image.c文件中的mono_image_open_from_data_with_name，代码如下：
```
MonoImage *

mono_image_open_from_data_with_name (char *data, guint32 data_len, gboolean need_copy, MonoImageOpenStatus *status, gboolean refonly, const char *name)

{

MonoCLIImageInfo *iinfo;

MonoImage *image;

char *datac;

if (!data || !data_len) {

if (status)

*status = MONO_IMAGE_IMAGE_INVALID;

return NULL;

}

    //红色部分是我们添加的，对加密过的dll进行解密

    if (strstr(name, “Assembly-CSharp.dll”) != NULL)

    {

        //这里是解密过程，我们采用的是xxtea加解密算法。

    }

return register_image (image);

}
```
## 1.2   Unity mono编译

有了加密方案，我们要解决的是mono编译问题。

- 1.2.1 源码下载

	https://github.com/Unity-Technologies/mono/tree/unity-4.5

- 1.2.2 编译Windows平台mono.dll

	1、打开Visual Studio Command Prompt(2010)

	2、进入mono-unity-4.5\msvc目录

	3、执行msbuild.exe mono.sln /p:Configuration=Release_eglib

	注意：直接打开mono.sln解决方案，在Visual Studio底下是编译不了的。

- 1.2.3 编译android平台libmono.so

	编译android下的libmono.so需要下载NDK工具链交叉编译。

	准备NDK r9以上版本。32位或者64位均可。

	Linux 32位环境下编译采用32位NDK版本。

	Linux 64位环境下编译采用64位NDK版本。

	Cygwin32位环境下编译采用32位NDK版本。

	Cygwin64位环境下编译采用64位NDK版本。

	建议使用32位linux与32位NDK来编译

	1、安装gcc、make、automake等编译工具

	$ sudo apt-get install build-essential

	Cygwin环境下安装devel里面的包，基本编译工具就都有了。

## 2、下载NDK，解压

	/home/jjl/android-ndk-r9

	设置环境变量export ANDROID_NDK_ROOT=/home/jjl/android-ndk-r9

## 3、下载mono源码，解压

	/home/jjl/mono-unity-4.5

## 4、安装依赖包

	bison

	gettext

	libffi-dev

	zlib（注意ubuntu底下包名为zlib1g-dev）

	libtool

	安装以上包用 sudo apt-get install xxxx

## 5、进入mono-unity-4.5目录

	编辑external/buildscripts/build_runtime_android.sh文件，这个文件是编译脚本

	找到第14行，填上对应的ndk版本号。

	perl ${BUILDSCRIPTSDIR}/PrepareAndroidSDK.pl -ndk=r9 -env=envsetup.sh && source envsetup.sh

	注释掉145、146行

	clean_build “$CCFLAGS_ARMv5_CPU” “$LDFLAGS_ARMv5″ “$OUTDIR/armv5″

	clean_build “$CCFLAGS_ARMv6_VFP” “$LDFLAGS_ARMv5″ “$OUTDIR/armv6_vfp”

	在mono-unity-4.5目录下执行

	./external/buildscripts/build_runtime_android.sh

	注意，一定要切换到mono-unity-4.5目录底下执行，而不要进入external/buildscripts/目录执行

	执行过程中，会出现以下错误：

	configure: error: C compiler cannot create executables

	编辑external/android_krait_signal_handler/build.pl这个文件

	将#!/usr/bin/env perl –w改为#!/usr/bin/perl –w

	其它的错误根源都因为这一行代码。

	再次进入mono-unity-4.5目录，执行

	./external/buildscripts/build_runtime_android.sh

	注意，一定要切换到mono-unity-4.5目录底下执行，而不要进入external/buildscripts/目录执行

	直到出现以下字样，说明我们已经编译成功了。

	Build SUCCESS!

	Build failed? Android STATIC/SHARED library cannot be found… Found 4 libs under builds/embedruntimes/android

	注意：出Build failed?字样，并不是我们编译失败了，而是我们注释掉了build_runtime_android.sh 第145、146行，有些库就没有编译。只编译我们要的库，加快编译速度。

	编译出来的库位于builds/embedruntimes/android/armv7a文件夹中。

	最后要编译release版，请将build_runtime_android.sh 第66行的-g选项去掉。