# 置顶机制调研与改进计划

## 结论

本项目当前使用 WPF `Window.Topmost`。它将窗口放入 Windows 的 topmost Z 顺序组，保证高于普通窗口；它不是跨越所有系统界面和窗口策略的“超级置顶”权限。Windows 没有面向普通桌面应用的受支持 API，可以保证窗口永远盖住全屏独占内容、安全桌面或所有其他置顶窗口。

不采用窃取或复制 UIAccess 令牌的实现。令牌代表进程安全身份，借用其他进程的令牌会绕过 Windows 安全边界。UIAccess 也不是通用置顶开关：微软将其限定给辅助技术客户端，并要求 Authenticode 签名、受信任来源、安全安装位置、manifest 中的 `uiAccess="true"` 及管理员启动条件。Ink Canvas 是绘图工具，除非产品正式满足辅助技术用途和部署要求，否则不应声明 UIAccess。

## 官方资料

- [WPF `Window.Topmost` 属性](https://learn.microsoft.com/en-us/dotnet/api/system.windows.window.topmost)：Topmost 窗口高于非 Topmost 窗口；同一组内仍由活动窗口决定顺序。
- [Win32 `SetWindowPos`](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-setwindowpos)：`HWND_TOPMOST` 将窗口放在所有非 Topmost 窗口之上；`SWP_NOACTIVATE` 可避免设置 Z 顺序时激活窗口。
- [辅助技术的安全注意事项](https://learn.microsoft.com/en-us/windows/win32/winauto/uiauto-securityoverview)：UIAccess 旨在让辅助技术跨进程交互，并明确列出签名、安全安装位置和 manifest 要求；不建议非辅助技术应用使用。
- [`SendMessage` 与 UIPI](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-sendmessage)：Windows 按完整性级别限制进程间的 UI 消息交互。它与窗口 Topmost Z 顺序是不同机制，提升 UIAccess 或管理员身份不等于获得普遍的置顶保证。

## 本项目普通置顶为何看起来失效

1. 主窗口在 [MainWindow.xaml](../Ink%20Canvas/MainWindow.xaml) 声明 `Topmost="True"`，但运行时状态会覆盖 XAML 初始值。
2. [MW_FloatBarIcons.cs](../Ink%20Canvas/MainWindow_cs/MW_FloatBarIcons.cs) 的 `BtnSwitch_Click` 在黑板/白板模式设置 `Topmost = false`，切回屏幕模式才设置 `Topmost = true`。这是当前设计，不是 Windows 忽略了置顶。工具栏布局代码也会在 `Topmost == false` 时隐藏浮动栏。
3. 模式状态分散在 `ImageBlackboard_Click`、`BtnSwitch_Click` 和 PowerPoint 放映处理器中。部分路径先改 `currentMode`，再调用其他方法间接同步 Topmost；如果路径、异常或未来修改漏掉同步，模式状态和实际窗口层级会不一致。代码目前没有一个统一的置顶状态入口或状态诊断记录。
4. `Topmost=true` 只保证高于非 Topmost 窗口。另一个 Topmost 窗口可以排在它前面；全屏独占程序、安全桌面、系统界面及应用自身的窗口策略也不受普通 Topmost 保证覆盖。高完整性级别应用还会受到 UIPI 的 UI 消息限制。
5. 当前 manifest 使用 `asInvoker` 和 `uiAccess="false"`（[app.manifest](../Ink%20Canvas/app.manifest)）。这本身不会让 WPF Topmost 失效，也不建议为追求置顶而改成管理员或 UIAccess 身份。

## 实施计划

### 阶段一：明确预期并修正状态同步

- 定义屏幕批注、黑板/白板、PPT 放映、隐藏墨迹等模式分别应否置顶，并把现有行为整理成状态表。
- 将 `currentMode` 与置顶策略的更新收敛到一个方法；模式切换、PPT 开始/结束和工具栏入口只调用该方法，不再分别写 `Topmost`。
- 明确区分“模式要求的置顶”与“用户手动固定窗口”。如果用户固定状态需要跨模式保留，加入单独设置并定义与黑板模式规则的优先级。
- 在模式切换和放映边界记录期望模式、`Topmost` 值及调用来源，便于复现“失效”是策略切换还是 Z 顺序限制。

### 阶段二：提供受支持的增强置顶

- 保留标准置顶作为默认行为；可增加显式“重新置顶”命令，在用户触发或关键模式切换时调用 WPF `Topmost`，必要时再用 `SetWindowPos(HWND_TOPMOST, SWP_NOMOVE | SWP_NOSIZE | SWP_NOACTIVATE)` 与 HWND 层同步。
- 检查并记录 HWND 是否已创建、Win32 返回值和错误码。调用不应激活窗口，也不应使用周期性抢 Z 顺序的定时器，避免抢焦点和与其他软件反复争抢。
- 对“高于普通窗口”以外的诉求，清晰呈现系统边界；遇到独占全屏或安全桌面时提示用户切换窗口化/无边框窗口模式或使用系统支持的辅助工具。
- 只有产品被正式定位为辅助技术、满足微软 UIAccess 部署条件并完成安全审查时，才单独评估 UIAccess。不得从其他进程提取令牌、伪造令牌或通过调试权限借用高权限身份。

### 阶段三：验证矩阵

- 屏幕批注与黑板/白板模式反复切换，验证 `currentMode`、实际 `Topmost` 和工具栏可见性一致。
- PPT 放映开始、结束、放映中切换模式，以及自动隐藏/展开工具栏。
- 与普通窗口、其他 Topmost 窗口、管理员运行的应用、窗口化/无边框全屏/独占全屏程序并行验证。
- 多显示器、DPI 变化、休眠恢复和窗口句柄重建后验证置顶恢复行为。
- 检查增强置顶不会激活 Ink Canvas、窃取键盘焦点或造成持续 Z 顺序争抢。

## 验收标准

- 每种应用模式都有明确、可测试的置顶期望值；所有模式入口通过同一状态更新方法。
- 普通桌面窗口场景中，置顶按预期工作；“重新置顶”操作不改变前台焦点。
- 对其他 Topmost 窗口、独占全屏和安全桌面的边界有清楚说明，不承诺系统不支持的绝对置顶。
- manifest 保持 `uiAccess="false"`，不引入令牌操作或为置顶而提升到管理员权限。
