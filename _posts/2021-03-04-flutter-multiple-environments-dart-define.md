---
layout: default
title: "利用 Dart-Define 实现 Flutter 多环境配置"
---

# 利用 Dart-Define 实现 Flutter 多环境配置

## 背景

现在常规的项目基本都有 2 个及以上的环境。为区分这些环境就需要对应的配置文件，我们需要它们来控制 API 地址、应用名称、版本号等信息。

在 Flutter 1.17 中，Flutter 会将在编译前将 dart-defines 传递给 iOS、Android、macOS。  
因此，可以几乎在各个时刻调用预先定义的 dart-defines。

## 目标

- 一处配置，多端使用。
- 熟悉的配置模式。采用 `.env.*` 模式来提供配置。
- 不入侵项目代码。
- 没有第三方依赖。

## 来！开始倒腾吧！

先来看看 `--dart-define` 的官方说明

> `dart-define=<foo=bar>`  
> Additional key-value pairs that will be available as constants from the String.fromEnvironment, bool.fromEnvironment, int.fromEnvironment, and double.fromEnvironment constructors.

利用 `--dart-define` 可以预先定义常量

```
flutter run --dart-define=APP_NAME=example_app --dart-define=USER_NAME=WzDTj
```

### 配置太多怎么办？

配置太多当然就写到配置文件中去呀！

```
// .env.local
APP_NAME=example_app_local
USER_NAME=WzDTj
```

```
// .env.staging
APP_NAME=example_app_staging
USER_NAME=WzDTj
```

```
// .env.production
APP_NAME=example_app_production
USER_NAME=WzDTj
```

### 可以指定配置文件吗？

写好配置文件，自然希望可以在命令中指定该文件。

遗憾的是官方并没有提供相关方法。

Github 上有人提议希望支持 .env，可惜官方并不愿意。

既然官方不支持，那就自己写个脚本实现呗。

```
// flutter_tool.sh

#!/bin/bash

for i in "$@"
do
case $i in
    --model=*)
    MODEL="${i#*=}"
    ;;
    *)
        # unknown option
    ;;
esac
done

VARS=( $(cut -d ' ' -f1 .env.$MODEL) )

DART_DEFINES=""
for (( i = 0; i < ${#VARS[@]}; i++ )); do
    DART_DEFINES+=" --dart-define=${VARS[i]}"
done

flutter $1 $DART_DEFINES
```

运行脚本

```
./flutter_tool run --model=local
```

### 如何使用定义好的 `dart-define` 呢？

在项目代码中直接调用 `fromEnvironment` 方法。

```
// *.dart
const appName = String.fromEnvironment("APP_NAME", defaultValue: "default ppp name");
```

类似的还有：

- String.fromEnvironment
- bool.fromEnvironment
- int.fromEnvironment

### 怎么控制 iOS 项目配置的信息呢？

当我们用 Flutter 编译 iOS 端时，会在 `./ios/Flutter` 目录下生成两个文件 `Generated.xcconfig`、`flutter_export_environment.sh`。

其中我们会看到之前设置的 dart-define。

```
// Generated.xcconfig
// ...
DART_DEFINES=APP_NAME%3Dexample_app_local,USER_NAME%3DWzDTj
// ...
```

```
// flutter_export_environment.sh
// ...
export "DART_DEFINES=APP_NAME%3Dexample_app_local,USER_NAME%3DWzDTj"
// ...
```

先来说下思路。

我们将 `DART_DEFINES` 解析到某个 `.xcconfig` 中，再通过引入该文件来达到我们想要的目的。

首先需配置 iOS 项目的 Scheme
将以下脚本代码输入到 `Build -> Pre-actions`。

```
function urldecode() { : "${*//+/ }"; echo "${_//%/\\x}"; }

IFS=',' read -r -a define_items <<< "$DART_DEFINES"

for index in "${!define_items[@]}"
do
    define_items[$index]=$(urldecode "${define_items[$index]}");
done

printf "%s\n" "${define_items[@]}" > ${SRCROOT}/Flutter/DartDefines.xcconfig
```

这样就可以在编译前得到一份 `DartDefines.xcconfig` 文件。

然后，在 `Debug.xcconfig`、`Release.xcconfig` 中引入该文件。

```
// ./ios/Flutter/Debug.xcconfig

#include "Generated.xcconfig"
#include "DartDefines.xcconfig"
```

```
// ./ios/Flutter/Release.xcconfig

#include "Generated.xcconfig"
#include "DartDefines.xcconfig"
```

最后，按需修改下 `Info.plist` 就行了。

例如：控制应用名称。

```
// ./ios/Runner/Info.plist
// ...
<key>CFBundleName</key>
<string>$(APP_NAME)</string>
// ...
```

### 怎么控制 Android 项目配置的信息呢？

Flutter 会将所有的 `dart-define` 用逗号链接成一个字符串传递给 Gradle。

```
// ./android/app/build.gradle

println(project.property('dart-defines'));

// Flutter v1.17 Output: APP_NAME=example_app_local,USER_NAME=WzDTj
// Flutter v1.20 Output: APP_NAME%3Dexample_app_local,USER_NAME%3DWzDTj

// 解析该字符串
def dartDefines = []
if (project.hasProperty('dart-defines')) {
    dartDefines = project.property('dart-defines')
            .split(',')
            .collectEntries { entry ->
                def pair = URLDecoder.decode(entry, StandardCharsets.UTF_8.toString()).split('=')
                [(pair.first()): pair.last()]
            }
}
```

得到 `dartDefines` 后，便可按需配置了。

例如：定义一个字符串资源供 `AndroidManifest.xml` 调用。

```
// ./android/app/build.gradle

defaultConfig {
    // ...
    resValue "string", "app_name", dartDefines.APP_NAME
}
```

```
// ./android/app/src/main/AndroidManifest.xml

<manifest xmlns:android="http://schemas.android.com/apk/res/android" package="com.example.flutter_app">
   <application
        android:name="io.flutter.app.FlutterApplication"
        android:label="@string/app_name"
        android:icon="@mipmap/ic_launcher">
        // ...
    </application>
</manifest>
```

### 用 Android Studio 开发怎么办呢？

可以将上述脚本中 `DART_DEFINES` 变量打印出来，填入 `Run/Debug Configurations` 中的 `Additional arguments` 即可。

## 参考资料

- [[flutter_tools] Allow providing dart-defines to Android, iOS, macOS builds](https://github.com/flutter/flutter/pull/51714)
- [[flutter_tools] Allow to define --dart-define from system environment variables or from .env](https://github.com/flutter/flutter/issues/56792)
- [Flutter 1.17 — no more Flavors, no more iOS Schemas. Command argument that changes everything](https://itnext.io/flutter-1-17-no-more-flavors-no-more-ios-schemas-command-argument-that-solves-everything-8b145ed4285d)
- [Flutter 1.20 — What you should know before you upgrade your project that uses compile-time variables](https://medium.com/@tatsu.ukraine/what-you-should-know-before-you-upgrade-flutter-in-your-project-with-compile-time-variables-3ec3d2e9ba79)
