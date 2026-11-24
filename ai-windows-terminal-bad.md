Windows Terminal（WT）在 **1.24 及之前**确实一直没有 Kitty 键盘协议和 `modifyOtherKeys`；微软直到 **WT 1.25**（2026 年 3 月进 Preview，**2026 年 10–11 月刚推送 Stable/GA**，如 `1.25.2733.0` / `1.25.3131.0`）才终于合入了 Kitty 键盘协议（`compatibility.kittyKeyboardMode`）。
不过即便升到了 1.25，目前也处于“刚上线、坑还很多”的阶段（比如切 alternate screen 丢协议 flags 的 `#20763`、部分 CLI 启用 Kitty 模式后按键失灵或中文输入法异常），再加上中间套了一层 Zellij，整体体验依然很折磨。

### 一、为什么 Windows Terminal 这么难用？

本质上不是某一个 Bug 的问题，而是 **Windows 控制台历史架构 + TSF 输入法框架 + 终端多层转译** 三重叠加的结果：

1. **ConPTY 的“双轨制原罪”**
   类 Unix 终端从头到尾都是纯字节流（PTY + VT 转义序列）；而 Windows 长期是另一套体系——`ReadConsoleInput` 读取结构化的 Win32 `INPUT_RECORD`（包含虚拟键码、扫描码、修饰键、按下/抬起事件）。
   WT 和后端程序之间隔着 **ConPTY**（实质上是无头 `conhost.exe` 在中间做“VT ↔ Win32 控制台缓冲区/事件”的双向翻译）。
   在这个转译层里，标准 VT 序列无法表达的东西很容易被直接压平（比如 `Shift+Enter` 压成 `\r`）。而微软最初的解法不是跟进 xterm 或 Kitty，而是发明了一个仅限自家的 **`win32-input-mode`**（`CSI ? 9001 h`），把 Win32 `INPUT_RECORD` 编码成 `CSI Vk;Sc;Uc;Kd;Cs;Rc _` 塞进流里。结果就是跨平台 TUI、Linux/WSL 工具、或者终端复用器要么不认识它，要么得专门写 Windows/WT 的 hack 去解码。

2. **中文输入法（TSF）与终端模型天生八字不合**
   “切到中文输入法只能打英文”在 WT 里是一个高频顽疾，背后通常是几个原因并发：
   - **TSF 焦点与 `InputScope` 错乱**：WT 基于 XAML/WinUI 的 `TSFInputControl` 接管输入法。一旦遇到 `Alt+Tab` 切换窗口、切分屏/标签页、或者 ConPTY 内部子进程（尤其是 WSL）发生焦点事件变化，Windows 的 TSF 文本服务就容易和微软拼音/第三方输入法的状态机失步，出现“状态栏显示中文，实际锁定在半角英文/原始拼音”的情况。
   - **键盘协议与 IME 打架**：在 WT 1.25 开启 Kitty 键盘协议或部分程序开启 `win32-input-mode` 时，如果按键事件被抢先拦截或按“裸键+修饰键”走增强序列上报，很容易绕过 IME 的组合（composition）流程，直接把拼音字母吐进输入框。
   - **伪光标与重绘（候选框问题）**：现代 TUI（比如 `pi`、Claude Code）喜欢自己画假光标，IME 找不到真实的硬件光标坐标；即便上层 CLI 修了光标跟随，一旦套了 Zellij，Zellij 在中间接管并重绘虚拟屏幕，物理光标位置又会被洗掉，导致 IME 候选框乱飘或输入状态异常。

3. **Zellij 在 Windows 下的放大效应**
   Zellij 本身是按 POSIX 终端思维设计的，2026 年才在 `0.44` 加上原生 Windows 支持（底层同样调 ConPTY）。**WT（一层 ConPTY）+ Zellij（又一层 ConPTY 或 WSL 桥接）**，等于输入输出要经过两层伪终端转译和多次跨线程 IPC，性能开销、包分片以及键盘协议桥接（Kitty ↔ Legacy）的 Bug 都会被成倍放大。

### 二、为什么微软改得这么慢 / 仿佛“不愿意改”？

与其说不想改，不如说是**历史包袱太重 + 路线优先级错位**：

- **必须给几十年 Win32 遗产兜底**：Kitty、Ghostty、Alacritty 只需要做好一个纯粹的 VT 终端模拟器；但 WT 和 `conhost.exe` / Windows 系统自带的 Inbox ConPTY 深度耦合，改任何输入输出管线都要保证几十年前的 `cmd`、PowerShell、老式 Win32 控制台程序、甚至无障碍读屏（UIA）不崩溃。微软长期把重心放在维护 `win32-input-mode` 和 ConPTY 内部一致性上，能不碰老结构就不碰。
- **工程文化里的路径依赖**：微软终端团队体量其实不大，早期对 Kitty 键盘协议一直偏保守（哪怕在 1.25 的官方更新日志里，微软都还在用 _"a highly opinionated way"_ 来形容 Kitty 协议）。他们一度觉得自家 `win32-input-mode` 信息量更全，是别的软件不适配 Windows；直到最近两年 AI Coding CLI（Claude Code、Codex、OpenCode、pi 等）爆发，整个跨平台 TUI 生态全面倒向 Kitty 协议，倒逼微软在 2026 年才终于把 Kitty 协议合进 1.25。
- **IME 问题跨部门扯皮**：很多“切窗口/切标签输入法变英文”的锅横跨 WT 的 UI 控件和 Windows 系统的 TSF/新版微软拼音团队。在微软内部，系统级输入法修复往往要跟着 Windows 版本走，周期极长。
