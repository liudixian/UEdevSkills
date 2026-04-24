---
name: "ue-plugin-rebuild"
description: "清理并重建 UE 项目/插件，并排查 UE C++ 编译异常。修改 C++ 后旧逻辑仍生效、出现 patch_*.exe、或报 generated.h/UHT 异常时调用。"
---

# UE 插件清理重建（确保改动生效）

## 适用场景（触发条件）

- 代码已修改但运行仍打印旧日志/旧错误文本
- 编辑器端（PIE）仍在使用旧的 `UnrealEditor-<Plugin>.dll`
- 发生过 Live Coding / Hot Reload，目录里出现 `UnrealEditor-*.patch_*.exe`
- 新增/删除了 .h/.cpp、改了 Build.cs/uplugin、或怀疑 UBT 增量构建判断失效
- 报错表面落在 `*.generated.h`、`UClass`、`DECLARE_CLASS`、`Z_Construct_UClass_*`，但头文件看起来又“没有问题”
- 只在某一个 `.cpp` 编译单元里失败，而同模块其他 `UCLASS` 正常

## 优先策略（从轻到重）

1. **仅逻辑改动（最常见）**：直接编译 **Editor 配置**（如 `ProjectNameEditor Win64 Development`）
2. **仍不生效**：执行“插件级清理重建”（删除插件 Binaries/Intermediate + 重建 Editor）
3. **仍不生效**：关闭编辑器后全工程清理（项目级 Intermediate/Binaries）再重建（谨慎）
4. **若报 `generated.h` / `UClass` 语法错误**：不要只盯着生成文件，转为做“对应头/源文件 + 编译单元环境”排查

## 插件级清理重建（推荐兜底流程）

### 0) 关闭 Unreal Editor

- 必须确认 UE 编辑器进程已退出，否则 DLL 可能被占用或仍在内存中生效

### 1) 删除插件构建产物

删除（目录以你的插件路径为准）：

- `Plugins/<PluginName>/Binaries`
- `Plugins/<PluginName>/Intermediate`

重点：清掉所有 `UnrealEditor-<PluginName>.patch_*.exe/.pdb/.lib` 等 Hot Reload 产物。

### 2) 重新生成项目文件（.sln）

使用 UBT 生成项目文件（示例）：

```powershell
& "<UE_5.x>\\Engine\\Build\\BatchFiles\\Build.bat" -ProjectFiles -project="<Project>.uproject" -game -engine
```

说明：
- 有些引擎安装不提供 `GenerateProjectFiles.bat`，这时用 `Build.bat -ProjectFiles` 是通用替代方案。

### 3) 重新编译 Editor（确保刷新 UnrealEditor-*.dll）

```powershell
& "<UE_5.x>\\Engine\\Build\\BatchFiles\\Build.bat" <ProjectName>Editor Win64 Development -Project="<Project>.uproject" -WaitMutex -FromMsBuild -architecture=x64
```

说明：
- **必须编译 Editor Target**，否则只编 `DebugGame/Development` 可能不会刷新编辑器端插件 DLL。

## `generated.h` / `UClass` 异常排查经验

### 先判断是不是“假根因”

- 如果错误形如 `error C2144`、`error C4430`，位置在 `SomeClass.generated.h` 里的 `UClass* Z_Construct_UClass_*`，通常 **报错位置不是真正根因**
- 先对比同模块其他正常类的 `.h` 与 `.generated.h`，确认生成代码结构是否一致
- 若 `generated.h` 内容与其他类一致，则优先怀疑：
  - 对应 `.cpp` 的编码/BOM 污染
  - 自头文件 `#include` 大小写不一致
  - 某个宏或前置声明在该编译单元被污染
  - 编译日志里还有更早的第一条错误被忽略

### 高价值排查顺序

1. **看 UBT 第一条真实错误**
   - 不要只看 IDE 红线，直接看 `Build.bat` 输出或 `UnrealBuildTool/Log.txt`
2. **对比正常类**
   - 和同模块其他正常 `UCLASS` 头文件对比 `#pragma once`、`*.generated.h` 位置、`UCLASS()`、`GENERATED_BODY()`、前置声明
3. **检查 `.cpp` 自头文件引用**
   - 让 `#include "MyClass.h"` 与磁盘文件名大小写完全一致
   - Windows 文件系统大小写不敏感，但 UBT/UHT/工具链缓存链路里不一致会制造隐蔽问题
4. **检查文件编码与 BOM**
   - 重点检查失败的 `.cpp` 开头是否出现多重 BOM、隐藏字符、异常前缀
   - 如果文件头有多个 `EF BB BF`，可能让编译单元上下文异常，最终把错误投射到 `generated.h`
5. **确认只在单编译单元失败还是全模块失败**
   - UBT 日志里如果明确是 `[1/4] Compile [x64] SomeClass.cpp` 失败，说明应重点看这个 `.cpp` 的直接输入环境
6. **必要时检查强制包含环境**
   - 查看 `Intermediate/.../*.obj.rsp`
   - 确认 `/FI` 的 `Definitions.<Module>.h` 中模块导出宏是否正常，例如 `<MODULE>_API`

### 本次案例经验

- 现象：
  - `FZbase.generated.h` 报 `INTERACTIONTOY_API UClass* Z_Construct_UClass_AFZBase_NoRegister();` 语法错误
  - 其他类的 `generated.h` 结构正常
- 真正根因：
  - `FZbase.cpp` 文件头存在异常 BOM 污染
  - `FZbase.cpp` 里引用写成了 `#include "FZBase.h"`，与实际文件 `FZbase.h` 大小写不一致
- 修复方式：
  - 规范化 `FZbase.cpp` 头部编码
  - 把自头文件 include 改为与磁盘一致的 `FZbase.h`
  - 再次执行 Editor Target 编译后恢复成功

### 排查注意事项

- 不要轻易直接修改 `Intermediate/.../*.generated.h`
- 不要因为 `Build.bat` 某次返回成功就立刻下结论，要确认失败的 `.cpp` 是否真的参与了本轮编译
- 如果日志里出现 `[Adaptive Build] Excluded from ... unity file: SomeClass.cpp`，说明该文件可能会独立编译，单文件问题更容易暴露
- 清理 `Intermediate/Binaries` 能解决缓存类问题，但 **不能替代** 源文件编码、include 路径、大小写的一致性检查

## 推荐执行清单

- 关闭 Unreal Editor，避免 DLL/中间文件被占用
- 优先编译 `<ProjectName>Editor Win64 Development`
- 如果仍失败，删除插件级或项目级 `Binaries`、`Intermediate`
- 使用 `Build.bat -ProjectFiles` 重新生成工程文件
- 重新构建 Editor Target
- 如果错误落在 `generated.h`，立刻检查对应 `.cpp` 的编码/BOM、include 大小写和日志第一条错误
- 修复后再次完整编译，不要只依赖 IDE 诊断

## 快速验证要点

- 检查 `Plugins/<PluginName>/Binaries/Win64/UnrealEditor-<PluginName>.dll` 的更新时间是否变更
- 重新打开 UE Editor 后再 PIE 验证日志是否为新行为
- 若蓝图里 PrintString 是常量文本，可能造成“看起来仍是旧逻辑”的误判；优先确认打印来源是函数输出口（OutError）
- 若是编译问题，确认构建输出最终出现 `Result: Succeeded`
