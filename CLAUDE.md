# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目定位

`PDMAddInDebugger` 是用于调试 SOLIDWORKS PDM Professional Add-In 的 C# 控制台宿主（host）。它本身**不是被调试的项目**，而是被外部 PDM Add-In 工程在 VS 的「调试 → 启动外部程序」里指向、作为承载 `Console.WriteLine` 输出流的壳。

主要使用场景（README 已说明）：把这个 exe 设成插件工程的启动外置程序，让插件中的 `Console.WriteLine` 直接落到本进程控制台，避免依赖 `explorer.exe` / `notepad.exe` 作为加载宿主。

## 技术栈与项目约束

- 语言：C#
- 框架：**.NET Framework v4.5.2**（`TargetFrameworkVersion=v4.5.2`，`supportedRuntime sku=".NETFramework,Version=v4.5.2"` 必须与 csproj 保持一致）
- MSBuild ToolsVersion：14.0（VS 2015 风格）
- 平台：AnyCPU，Debug 配置 `Prefer32Bit=false`
- 输出：`PDMAddInDebugger/bin/{Debug,Release}/PDMAddInDebugger.exe`
- 唯一外部 UI 引用：`System.Windows.Forms`（仅用到 `OpenFileDialog`），不需要 WPF
- 没有 NuGet 包引用，不需要 `nuget restore`；没有项目间引用，不需要 `AssemblyInfo` 之外的额外源

仓库级约束（来自 `.gitattributes`）：
- `*.cs / *.vb / *.bat / *.ps1 / *.config / *.resx` 强制 CRLF
- 其他文本默认 LF（`* text=auto eol=lf`）
- 二进制（`.dll / .exe / .pdb / .zip / 图片` 等）按 binary 处理，不做换行转换

修改代码时不要强行改换行符；否则 PR diff 会被空白字符撑大。

## 构建

```bash
msbuild PDMAddInDebugger.sln /p:Configuration=Debug
# 或
msbuild PDMAddInDebugger/PDMAddInDebugger.csproj /p:Configuration=Debug
```

Release 同理换 `Configuration=Release`。产物路径：`PDMAddInDebugger/bin/Debug/PDMAddInDebugger.exe`。

VS 用户：打开 `PDMAddInDebugger.sln`，生成解决方案即可，不需要额外还原步骤。

## 运行 / 使用流程（本仓库不出单元测试）

本仓库**没有测试框架、lint、CI**——它就是一个几十行的壳，不存在 `dotnet test` / CI workflow。任何「验证」都按 README 的工作流手工完成：

1. 在 PDM Add-In 工程里，把 `PDMAddInDebugger.exe`（release 包或自己 `bin\Debug\` 产物）设为「启动外部程序」。
2. （可选）命令行参数传入 PDM vault view 的本地路径，作为 OpenFileDialog 的初始目录。
3. 启动后会看到黄色 banner 和启动时间戳，弹出多选文件对话框（`Filter=All files (*.*)|*.*`，`Multiselect=true`）；选完后控制台停在那里，按 Enter / 任意键结束，期间插件内任何 `Console.WriteLine` 都直接落到这个控制台。

## 代码结构（核心就一个文件）

```
PDMAddInDebugger.sln
PDMAddInDebugger/
├── Program.cs                # 唯一业务逻辑：[STAThread] static void Main(args)
├── PDMAddInDebugger.csproj   # 输出 bin\Debug / bin\Release
├── App.config                # supportedRuntime = .NETFramework v4.5.2
└── Properties/AssemblyInfo.cs  # AssemblyVersion / FileVersion = 1.1.0.0
```

`Program.Main` 的关键节点（按执行顺序）：

1. 保存 `Console.ForegroundColor`，改黄输出 banner（`Amen Jlili, amen@bluebyte.biz - <ProductName> <ProductVersion>`）。
2. 从 `args[0]` 读取起始目录：空 → 红字提示后回退到 `Environment.SpecialFolder.Desktop`；非空但 `Directory.Exists` 为 false → 打印报错、`Console.ReadKey()` 后 `return`。
3. 打印分隔线，弹出 `OpenFileDialog`（`InitialDirectory = directory`）。
4. 用户确认后再打一行分隔线 + `Console.ReadLine()` 等待停顿，最后 `Debugging ended at <now>` + `Console.ReadKey()`。

注意事项：
- `[STAThread]` 不可去掉——`OpenFileDialog` 强制要求 STA 线程。
- 实际加载插件 DLL / 触发 `IEdmAddIn5` 钩子的逻辑**不在本仓库代码内**。本进程只负责提供 `Console` 输出目标，DLL 由 PDM Professional 注册表机制或用户对 `OpenFileDialog` 结果自行处理。
- `Application.ProductVersion` 来自 `AssemblyInfo.FileVersion`，改版本号要同步两边。

## 修改与扩展时的几个容易踩的点

- **不要改 `TargetFrameworkVersion` 之外加新包**。NuGet 引入会触发首次 `packages.config` 提交，仓库现在刻意回避这个负担。
- **不要把 `Program.cs` 重构成多文件 / 多 class**。当前是单文件单方法，整库 60 行以内，重构会破坏「最少代码完成一件事」的初衷。
- **如果要改 banner / 公司信息**，那是厂商署名（`Blue Byte LLC`，原作者 `Amen Jlili, amen@bluebyte.biz`），是事实信息不是技术实现，修改前必须先与仓库所有者确认。
- 新增诊断输出请用 `Console.ForegroundColor = ConsoleColor.Red/Yellow + WriteLine` + 还原原 `ForegroundColor` 的同款模式，不要直接污染默认颜色。
