# GTOdyssey "seal" — 逆向结论

## 一、护盾的真实结构（由字节码 + 运行时探针确认）

`gtocore-forge-1.20.1-26.8.3-techtree.jar` 使用的是一套 **ModLauncher 深度集成的字节码保护**
（seal runtime = `gto-seal-runtime-1.0`，原生库 = `native0/x64-windows.dll`，MIT，作者 Su5eD）。

三部分组成：

| 组件 | 作用 |
| --- | --- |
| `native0/Loader` | 按 `os.name`/`os.arch` 选库，复制到临时文件后 `System.load` |
| `gto.native0.plugins.GTOServices` | `ITransformationService`，整类原生实现（`initialize`/`onLoad`/`transformers`/`completeScan`） |
| `gto.native0.plugins.GTOPlugin` | `ILaunchPluginService`，对 `com.gtolib.*` 返回 `Phase.BEFORE` |

关键调用链：

```
ModLauncher 变换 com.gtolib.X
  → GTOPlugin.processClassWithFlags(...)          // 只把 Type 交给原生
    → GTOProvider.init_gtolib(org.objectweb.asm.Type)
      → LoadingModList.get()                      // ← 需要真实 Forge 环境
      → 解密 native0/native/<name>.prod.bin
```

`GTOProvider` 持有两个 `cpw.mods.jarhandling.SecureJar$ModuleDataProvider` 静态字段，
说明解密结果是**通过 SecureJar 的 provider 提供的**，而不是走 JVMTI 注入。

原生方法注册情况（运行时 `RegisterNatives` 实测）：

- `GTOProvider`：`ㅤࣦ࣮ࣶ(Object,Object)I`、`init_gtolib(Type)V`、`init_gtolib(ClassLoader)V`、`decrypt(byte[],String)V`
- `GTOServices`：8 个（含 `ㅤࣦ࣮ࣶ(LinkedHashMap,ILaunchPluginService)V`、`ㅤࣦ࣮ࣶ(String)Z`）
- `native0/Loader.registerNativesForClass(I,Class)V`、`Hidden0.special_clinit_N_M(Class)V`

注意：这些混淆名三个类里都相同，均为 `U+3164` + 三个阿拉伯组合符（`e3 85 a4 e0 a3 a6 e0 a3 ae e0 a3 b6`）。

## 二、`.prod.bin` 格式（已知明文对照）

675 个 `native0/native/**.prod.bin`，JAR 中 **STORE 存放**：

- 熵 **7.874**，接近纯随机
- 首/尾 32 字节在 675 个文件间 **几乎无重复**（每个位置约 237~249 种取值）→ **没有固定 magic/header**
- 长度模 16 分布均匀 → **不是分组密码**（流密码或自定义逐字节）
- 无重复 16 字节块 → 不是 ECB 复用
- `prod.bin` 普遍**比明文类更大**（Client 1452 → 类 920；GTAddon 8579 → 1465），
  说明解密后的明文是一个**容器**（比类大），而不是裸类

结论：需要**密钥/算法**才能解，无法靠结构推断。

## 三、为什么进程内调用原生解密失败（已定位到具体指令）

`init_gtolib(org.objectweb.asm.Type)` 必然抛
`NullPointerException: "INVOKEVIRTUAL Object npe" on 1067`。

用 JNI 表插桩（hook 槽位 33/36/116/113）后确认原因：

```
[JNI] GetStaticMethodID get ()Lnet/minecraftforge/fml/loading/LoadingModList; on LoadingModList;
→ 返回 null（单例未初始化）
→ 原生代码对 null 调用方法 → NPE
```

尝试修复：用 JNI 字段写入（`SetStaticObjectField` 绕过访问控制）强行种入空 `LoadingModList`。
结果 `LoadingModList.of()` 自身抛 `NoClassDefFoundError: net/minecraftforge/forgespi/locating/IModFile`
——缺 Forge SPI，装完还会缺下一个。

**因此：seal 的解密路径只能在真实 Forge/ModLauncher 启动流程中运行。**
裸 JVM 里手工拼装 Forge 运行时是死路。

## 四、一个意外但重要的事实

这个 `techtree` jar **同时包含明文类和密封副本**：

- `com/gtolib/**.class`（837 个，明文）—— 其中 675 个与 `*.prod.bin` 一一同名对应
- `native0/native/**.prod.bin`（675 个，密封）

逐字节比对：**674/675 完全一致**（唯一差异是 `com/gtolib/Client`，
因为测试时被 69 字节占位类覆盖过 `jvmtiout` 里的副本）。

也就是说：**这个 jar 的明文类 = 加载器实际服务的内容**。
如果目标只是阅读该 mod 的代码，直接解压即可，不需要解密。

反过来，如果真正的目标是**被剥离明文的发布版 jar**，那必须走下面两条路之一。

## 五、可选的后续路线

- **A. 真实启动 ModLauncher**：用实例 `D:\Software\lunalauncher\instances\GregTech.Odyssey-0.5.6-beta\minecraft`
  的真实启动参数跑一次，配合 Zig 编译的 **JVMTI agent**（`-agentpath:`）在 `ClassFileLoadHook`
  里把每个被定义的类写盘。最忠实，但需要能启动游戏。
- **B. 静态逆向 DLL**：`sub_1801094d4.bin`（129,932 B，在 `新增資料夾\`）、`full.asm`、`sx8.asm`、
  `dis.exe`(capstone 5.0.7) 都在。但该函数是**垃圾字节插入**的（开头即是真实序言与噪声交织），
  且 IAT 655 项**符号名全部被抹空**，是硬骨头。
- **C. 直接产出明文类**：若当前 jar 足够，直接抽取 837 个 `com/gtolib/*.class` 并反编译。

## 六、本次产出的工具能力

`gtodump/gtodec.zig`（Zig，约 1600 行，`zig build-exe gtodec.zig -target x86_64-windows -O ReleaseSafe`）：

- 进程内 `JNI_CreateJavaVM` 启动真实 JVM（Zulu 17）
- 挂接 JNI 函数表（槽位 6/33/36/113/116/184/192/200/208/215）做调用追踪与 `byte[]` 影子拷贝
- 安装 JVMTI `ClassFileLoadHook`（vtable 索引 46/47/77/121/141/151），可 `RetransformClasses`
- Unicode 安全落盘（`CreateFileW` + UTF-8→UTF-16）
- `JGT_MODE=probe`：13 步依次驱动 seal 的每个原生入口并逐窗口抓取
- 批量模式：按名单 `FindClass` 并落盘所有密封类
