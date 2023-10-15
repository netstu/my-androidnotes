# 对现有的 Zygisk 实现方案的整理

[原版 (这里选择 v25.2)](https://github.com/topjohnwu/Magisk/blob/6066b5cf86703512451a021cf1aaf1a877530af7/native/jni/zygisk)

[maru (在 v25.2 基础上的)](https://github.com/5ec1cff/Magisk/tree/maru)

[ZygiskOnKernelSU](https://github.com/Dr-TSNG/ZygiskOnKernelSU)

[nbzygisk](https://github.com/LSPosed/Metagisk/tree/zygisk)

## 加载和 JNI hook

```cpp
app_process main()
AndroidRuntime::setArgv0
AndroidRuntime::start
 AndroidRuntime::startVm
  JNI_CreateJavaVM
   art::Runtime::Create
    art::Runtime::Init
     art::Runtime::LoadNativeBridge # NB Loaded
   art::Runtime::Start
     Runtime::InitNativeMethods # register native methods of framework
 AndroidRuntime::onVmCreated <- original zygisk / maru
 AndroidRuntime::startReg
  androidSetCreateThreadFunc <- ZygiskOnKernelSU
 AndroidRuntime::toSlashClassName
  strdup (com.android.internal.os.ZygoteInit) <- nbzygisk
```

原版 zygisk 方案：hook setArgv0 拿到 AndroidRuntime 对象，虚表 hook onVmCreated

maru 方案：加载 NB 后 `AndroidRuntime::getRuntime` 获取 AndroidRuntime ，虚表 hook onVmCreated

ZygiskOnKernelSU 方案：hook androidSetCreateThreadFunc ，通过 GetJavaVM 取得 JavaVM 和 JNIEnv

nbzygisk 方案：

### nbzygisk 的加载

nb stub: 

https://github.com/LSPosed/Metagisk/blob/0eb0fc9551067425e5cd470d347e99b86f0b2206/native/src/zygisk/native_bridge.cpp#L25

和 riru 方案不同。riru 是指定版本 0 ，直接 dlclose ，而 nbzygisk 指定版本 2 ，实现了 isCompatibleWith 并返回 false 。

https://cs.android.com/android/platform/superproject/main/+/main:art/libnativebridge/native_bridge.cc;l=271;drc=c5cf4482ff754b16b3f90717ee9b1c6047876f52

在 isCompatibleWith 中加载 hook lib 。hook lib 会 hook libnativebridge.so 的 dlclose ，通过 Unwind 和内存 / 寄存器上下文搜索找到 LoadNativeBridge 和 NativeBridgeRuntimeCallbacks ，如果有原始 nb ，则重新调用 LoadNativeBridge 。由于是在 dlclose 处重入了 LoadNativeBridge ，可以看到新 nb 加载后 NativeBridge 的内部状态不再是加载了假 nb lib 的异常。

https://github.com/LSPosed/Metagisk/blob/0eb0fc9551067425e5cd470d347e99b86f0b2206/native/src/zygisk/hook.cpp#L198

此外 nbzygisk 还要利用 NativeBridgeRuntimeCallbacks 进行 JNI hook

https://cs.android.com/android/platform/superproject/main/+/main:art/libnativebridge/include/nativebridge/native_bridge.h;l=427;drc=c5cf4482ff754b16b3f90717ee9b1c6047876f52

https://github.com/LSPosed/Metagisk/blob/0eb0fc9551067425e5cd470d347e99b86f0b2206/native/src/zygisk/hook.cpp#L222

直接利用 getNativeMethods 获取相应类的 JNI 原函数指针，然后 `env->RegisterNatives` 注册新的，这样就不需要替换 env RegisterNatives ，也不需要自己建立一个 map 维护所有的 native 方法了。

不过西大师的 Unwind 还是过于魔法，我等既学不会也不敢用。

## forkAndSpecialize

```cpp
com.android.internal.os.Zygote.forkAndSpecialize
 com_android_internal_os_Zygote_nativeForkAndSpecialize
  zygote::ForkCommon
   __android_log_close
   FileDescriptorTable::Create || gOpenFdTable->Restat
    FileDescriptorInfo::CreateFromFd # fd check
   fork
  zygote::SpecializeCommon
   ensureInAppMountNamespace (API 30+ always) || MountEmulatedStorage
    unshare (CLONE_NEWNS)
   selinux_android_setcontext
 davik.system.ZygoteHooks.postForkCommon
  java.lang.Daemons.startPostZygoteFork
   ...
   Thread.start -> Thread.nativeCreate
    art::Thread::CreateNativeThread
     pthread_attr_destroy
```

zygisk 为了实现模块与 zygote 隔离，在 forkAndSpecialize 的时候提前 fork ，在子进程加载模块，两个进程都执行 pre ForkCommon 逻辑，等到正式调用 fork 才返回预先 fork 好的 pid 。

### fd 检查

TODO

### unshare

TODO

###

### 同步 dlclose

选择一个和 dlclose 返回值和参数类型完全相同（即签名相同）的函数 hook 。这个函数要在 specialize 完成后调用一次。这里选择了 pthread_attr_destroy ，是 art 启动 daemon 线程的时候调用，其返回值正常的情况下为 0 ，和 dlclose 一致。

签名相同的函数可以利用 `[[clang::musttail]]` ，让 dlclose 复用 pthread_attr_destroy 的栈帧，dlclose 返回的时候就不会回到我们的被卸载的代码，而是回到调用 pthread_attr_destroy 的代码，实现同步 dlclose 。

这个方法也是西大师提出的，并首先在 nbzygisk 中实现。

https://clang.llvm.org/docs/AttributeReference.html#musttail

考虑以下代码

```cpp
[[clang::optnone]]
int a(int x) {
  int y = x + 1;
  return y;
}

[[clang::optnone]]
int b(int x) {
  int y = x + 1;
  [[clang::noinline]] [[clang::musttail]] return a(y);
}

int main() {
  [[clang::noinline]] return b(100);
}
```

反汇编(aarch64)：

```
0000000000000640 <_Z1ai>:
     640: d10043ff      sub     sp, sp, #0x10
     644: b9000fe0      str     w0, [sp, #0xc]
     648: b9400fe8      ldr     w8, [sp, #0xc]
     64c: 11000508      add     w8, w8, #0x1
     650: b9000be8      str     w8, [sp, #0x8]
     654: b9400be0      ldr     w0, [sp, #0x8]
     658: 910043ff      add     sp, sp, #0x10
     65c: d65f03c0      ret

0000000000000660 <_Z1bi>:
     660: d10043ff      sub     sp, sp, #0x10
     664: b9000fe0      str     w0, [sp, #0xc]
     668: b9400fe8      ldr     w8, [sp, #0xc]
     66c: 11000508      add     w8, w8, #0x1
     670: b9000be8      str     w8, [sp, #0x8]
     674: b9400be0      ldr     w0, [sp, #0x8]
     678: 910043ff      add     sp, sp, #0x10
     67c: 17fffff1      b       0x640 <_Z1ai>
```

如果注释掉 musttail：

```
0000000000000670 <_Z1bi>:
     670: d10083ff      sub     sp, sp, #0x20
     674: a9017bfd      stp     x29, x30, [sp, #0x10]
     678: 910043fd      add     x29, sp, #0x10
     67c: b81fc3a0      stur    w0, [x29, #-0x4]
     680: b85fc3a8      ldur    w8, [x29, #-0x4]
     684: 11000508      add     w8, w8, #0x1
     688: b9000be8      str     w8, [sp, #0x8]
     68c: b9400be0      ldr     w0, [sp, #0x8]
     690: 97fffff0      bl      0x650 <_Z1ai>
     694: a9417bfd      ldp     x29, x30, [sp, #0x10]
     698: 910083ff      add     sp, sp, #0x20
     69c: d65f03c0      ret
```

可以发现，musttail 会把函数调用优化成一个跳转语句，之后没有任何代码。

[为何 musttail 限制签名相同的讨论](https://github.com/llvm/llvm-project/issues/54964)

实际上这个工作也可以通过手写汇编完成，不过关键还是要找到一个合适的 hook 点，其返回值为 0 。
