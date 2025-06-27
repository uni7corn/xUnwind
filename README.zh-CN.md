# xUnwind

![](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat)
![](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat)
![](https://img.shields.io/badge/release-2.0.0-red.svg?style=flat)
![](https://img.shields.io/badge/Android-4.1%20--%2013-blue.svg?style=flat)
![](https://img.shields.io/badge/arch-armeabi--v7a%20%7C%20arm64--v8a%20%7C%20x86%20%7C%20x86__64-blue.svg?style=flat)

xUnwind 是一个安卓 native 栈回溯方案的集合。

[README English Version](README.md)


## 特征

* 支持的栈回溯方案：
    * CFI (Call Frame Info)：由安卓系统库提供。
    * EH (Exception handling GCC extension)：由编译器提供。
    * FP (Frame Pointer)：只支持 ARM64。
* 支持的栈回溯起点：
    * 当前执行位置。
    * 指定的上下文（可能是从信号处理函数中获取）。
* 支持的栈回溯的进程：
    * 当前进程。
    * 其他进程：只支持 CFI 方案。
* 支持的栈回溯的线程：
    * 当前线程。
    * 指定的线程：只支持 CFI 方案。
    * 所有的线程：只支持 CFI 方案。
* 提供 java 函数，在 java 代码中直接获取 native 调用栈。
* 支持 Android 4.1 - 13（API level 16 - 33）。
* 支持 armeabi-v7a, arm64-v8a, x86 和 x86_64。
* 使用 MIT 许可证授权。


## 使用

### 1. 在 build.gradle 中增加依赖

xUnwind 发布在 [Maven Central](https://search.maven.org/) 上。为了使用 [native 依赖项](https://developer.android.com/studio/build/native-dependencies)，xUnwind 使用了从 [Android Gradle Plugin 4.0+](https://developer.android.com/studio/releases/gradle-plugin?buildsystem=cmake#native-dependencies) 开始支持的 [Prefab](https://google.github.io/prefab/) 包格式。

```Gradle
android {
    buildFeatures {
        prefab true
    }
}

dependencies {
    implementation 'io.github.hexhacking:xunwind:2.0.0'
}
```

**注意**：

1. xUnwind 从版本 `2.0.0` 开始，group ID 从 `io.hexhacking` 改为 `io.github.hexhacking`。

| 版本号范围          | group ID                 | artifact ID | 仓库 URL |
|:---------------|:-------------------------|:------------|:----------------------------------------------------------|
| [1.0.1, 1.1.1] | io.hexhacking            | xunwind     | [repo](https://repo1.maven.org/maven2/io/hexhacking/xunwind/) |
| [2.0.0, )      | **io.github.hexhacking** | xunwind     | [repo](https://repo1.maven.org/maven2/io/github/hexhacking/xunwind/) |

2. xUnwind 使用 [prefab package schema v2](https://github.com/google/prefab/releases/tag/v2.0.0)，它是从 [Android Gradle Plugin 7.1.0](https://developer.android.com/studio/releases/gradle-plugin?buildsystem=cmake#7-1-0) 开始作为默认配置的。如果你使用的是 Android Gradle Plugin 7.1.0 之前的版本，请在 `gradle.properties` 中加入以下配置：

```
android.prefabVersion=2.0.0
```

### 2. 在 CMakeLists.txt 或 Android.mk 中增加依赖

如果你只使用 xUnwind 的 java 接口，请跳过这一步。

> CMakeLists.txt

```CMake
find_package(xunwind REQUIRED CONFIG)

add_library(mylib SHARED mylib.c)
target_link_libraries(mylib xunwind::xunwind)
```

> Android.mk

```
include $(CLEAR_VARS)
LOCAL_MODULE           := mylib
LOCAL_SRC_FILES        := mylib.c
LOCAL_SHARED_LIBRARIES += xunwind
include $(BUILD_SHARED_LIBRARY)

$(call import-module,prefab/xunwind)
```

### 3. 指定一个或多个你需要的 ABI

```Gradle
android {
    defaultConfig {
        ndk {
            abiFilters 'armeabi-v7a', 'arm64-v8a', 'x86', 'x86_64'
        }
    }
}
```

### 4. 增加打包选项

如果你是在一个 SDK 工程里使用 xUnwind，你可能需要避免把 libxunwind.so 打包到你的 AAR 里，以免 app 工程打包时遇到重复的 libxunwind.so 文件。

```Gradle
android {
    packagingOptions {
        exclude '**/libxunwind.so'
    }
}
```

另一方面, 如果你是在一个 APP 工程里使用 xUnwind，你可能需要增加一些选项，用来处理重复的 libxunwind.so 文件引起的冲突。

```Gradle
android {
    packagingOptions {
        pickFirst '**/libxunwind.so'
    }
}
```

你可以参考 [xunwind-sample](xunwind_sample) 文件夹中的示例 app。


## Native API

```C
#include "xunwind.h"
```

### 1. CFI 栈回溯

```C
#define XUNWIND_CURRENT_PROCESS (-1)
#define XUNWIND_CURRENT_THREAD (-1)
#define XUNWIND_ALL_THREADS (-2)

void xunwind_cfi_log(pid_t pid, pid_t tid, void *context, const char *logtag, android_LogPriority priority, const char *prefix);
void xunwind_cfi_dump(pid_t pid, pid_t tid, void *context, int fd, const char *prefix);
char *xunwind_cfi_get(pid_t pid, pid_t tid, void *context, const char *prefix);
```

这三个函数对应了三种获取 backtrace 的方式。它们是：

* `log` 到 Android logcat。
* `dump` 到一个 FD 相关的地方（比如文件，管道，套接字，等等）。
* `get` 并返回一个字符串（由 `malloc()` 在堆中分配，你需要自己用 `free()` 释放它）。

`pid` 参数用于指定需要回溯哪个进程，可以是当前进程（`XUNWIND_CURRENT_PROCESS` / `getpid()`），或者是其他进程。

`tid` 参数用于指定需要回溯哪个或哪些线程，可以是当前线程（`XUNWIND_CURRENT_THREAD` / `gettid()`），某个指定的线程，或所有线程（`XUNWIND_ALL_THREADS`）。

可选的 `context` 参数用于指定寄存器上下文信息。比如，在信号处理函数中，你可以需要从某个特定的寄存器上下文开始执行栈回溯。

可选的 `prefix` 参数用于给所有的 backtrace 行指定一个字符串前缀。

### 2. FP 和 EH 栈回溯

```C
size_t xunwind_fp_unwind(uintptr_t* frames, size_t frames_cap, void *context);
size_t xunwind_eh_unwind(uintptr_t* frames, size_t frames_cap, void *context);

void xunwind_frames_log(uintptr_t* frames, size_t frames_sz, const char *logtag, android_LogPriority priority, const char *prefix);
void xunwind_frames_dump(uintptr_t* frames, size_t frames_sz, int fd, const char *prefix);
char *xunwind_frames_get(uintptr_t* frames, size_t frames_sz, const char *prefix);
```

目前，FP 栈回溯只支持 ARM64。

`xunwind_fp_unwind` 和 `xunwind_eh_unwind` 将栈回溯的结果（一组绝对 PC 值）保存到 `frames` 数组中（`frames_cap` 指定了数组的大小），并且返回获取到的绝对 PC 值的个数。

可选的 `context` 参数的作用和 CFI 栈回溯中相同。

剩下的三个函数用于将 `frames` 数组中保存的绝对 PC 值转换成 backtrace（数组的大小由 `frames_sz` 参数指定）。和 CFI 栈回溯相同，分别输出到 Android logcat，FD，或者返回一个字符串。


## Java API

```Java
import io.github.hexhacking.xunwind.XUnwind;
```

### 1. 初始化

```Java
public static void init();
```

`init()` 只做了 `System.loadLibrary("xunwind")` 这一件事。如果你只在 native 代码中使用 xUnwind 的话，就不需要初始化了。

### 2. CFI 栈回溯

```Java
public static void logLocalCurrentThread(String logtag, int priority, String prefix);
public static void logLocalThread(int tid, String logtag, int priority, String prefix);
public static void logLocalAllThread(String logtag, int priority, String prefix);
public static void logRemoteThread(int pid, int tid, String logtag, int priority, String prefix);
public static void logRemoteAllThread(int pid, String logtag, int priority, String prefix);

public static void dumpLocalCurrentThread(int fd, String prefix);
public static void dumpLocalThread(int tid, int fd, String prefix);
public static void dumpLocalAllThread(int fd, String prefix);
public static void dumpRemoteThread(int pid, int tid, int fd, String prefix);
public static void dumpRemoteAllThread(int pid, int fd, String prefix);

public static String getLocalCurrentThread(String prefix);
public static String getLocalThread(int tid, String prefix);
public static String getLocalAllThread(String prefix);
public static String getRemoteThread(int pid, int tid, String prefix);
public static String getRemoteAllThread(int pid, String prefix);
```

所有的 native CFI 栈回溯能力都有对应的 java 函数，它们通过 JNI 调用 native CFI 函数。

FP 和 EH 栈回溯没有对应的 java 函数。因为相对于 CFI 栈回溯，它们的主要优势是执行速度更快（但是 backtrace 并不像 CFI 栈回溯那样完整），所以它们总是只在 native 代码里使用。如果你是 java 程序员，只使用这里的 CFI 栈回溯就可以了。


## 技术支持

* [GitHub Issues](https://github.com/hexhacking/xUnwind/issues)
* [GitHub Discussions](https://github.com/hexhacking/xUnwind/discussions)


## 贡献

* [Code of Conduct](CODE_OF_CONDUCT.md)
* [Contributing Guide](CONTRIBUTING.md)
* [Reporting Security vulnerabilities](SECURITY.md)


## 许可证

xUnwind 使用 [MIT 许可证](LICENSE)。
