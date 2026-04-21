# Windows 下 Codex 命中内置 rg 但实际不可用的排查与修复

## 1. 问题现象

在 Windows 下使用 Codex 时，可能会出现一种很迷惑的情况：

- 手动输入 `rg --version` 看起来能正常输出版本号
- 但 Codex 执行搜索时仍然提示没有可用的 `rg`
- 或者会退回到 PowerShell 原生搜索
- 或者命中的是 `WindowsApps` 里的 `OpenAI.Codex_...\\app\\resources\\rg.exe`

这类问题的关键不是“机器有没有安装 ripgrep”，而是：

> Codex 实际命中的 `rg.exe` 是否是机器上独立可访问的版本。

## 2. 根因

Windows 上的 Codex 桌面版可能自带一份内置 `rg.exe`，路径类似：

```text
C:\Program Files\WindowsApps\OpenAI.Codex_...\app\resources\rg.exe
```

它在某些终端里可能能执行 `rg --version`，但对 Codex 工具链来说并不稳定，常见问题包括：

- 工具调用时访问被拒绝
- 只能看到内置路径，看不到机器上独立安装的 ripgrep
- 当前会话与用户终端的 PATH 解析结果不一致

所以要特别注意：

> `rg --version` 能跑，不等于 Codex 工具链就能正常使用 `rg`。

## 3. 正确判断方式

不要只看版本号，先看实际命中了哪一个 `rg.exe`。

### 3.1 查看当前命令解析结果

```powershell
Get-Command rg
```

更直接一点：

```powershell
(Get-Command rg).Source
```

### 3.2 查看系统里所有可见的 `rg`

```powershell
where.exe rg
```

### 3.3 理想结果

优先命中的应该是机器上独立安装的 ripgrep，例如：

```text
C:\tools\ripgrep\rg.exe
```

或：

```text
C:\Program Files\PowerShell\7\rg.exe
```

或其他明确属于用户/系统工具目录的路径。

### 3.4 需要警惕的结果

如果命中的是类似下面的路径：

```text
C:\Program Files\WindowsApps\OpenAI.Codex_...\app\resources\rg.exe
```

就要把它当成 Codex 内置二进制，而不是机器上的首选 `rg`。

## 4. 推荐修复思路

### 4.1 原则

目标不是“让任何一个 `rg` 都能跑”，而是：

- 安装一份机器上独立可访问的 ripgrep
- 让 PATH 优先命中这份独立版
- 让 Codex 和普通 PowerShell 都解析到同一份稳定路径

### 4.2 可行做法

1. 安装独立版 ripgrep
2. 把 `rg.exe` 放到稳定、长期存在、可访问的目录
3. 确保该目录在 PATH 里优先级足够靠前
4. 重新打开终端 / 重新打开 Codex

### 4.3 一种稳定方案

如果机器本身长期使用 PowerShell 7，而且 `C:\Program Files\PowerShell\7` 已经在 PATH 前列，那么把独立版 `rg.exe` 放到这里通常比较稳：

```text
C:\Program Files\PowerShell\7\rg.exe
```

这样做的好处是：

- 路径稳定
- 普通 PowerShell 能直接命中
- Codex 工具环境通常也更容易访问到

## 5. 验证步骤

修复后至少验证这三件事：

### 5.1 版本可执行

```powershell
rg --version
```

### 5.2 当前解析路径正确

```powershell
(Get-Command rg).Source
```

### 5.3 实际搜索可执行

```powershell
rg --files C:\github\ai-knowledge-base
```

如果这三步都正常，再看 Codex 是否还会提示“没有可用的 rg”。

## 6. 如果仍然不稳定怎么办

如果机器上已经有独立版 `rg.exe`，但 Codex 仍然优先命中内置路径，可以采用更保守的执行策略：

1. 先用 `(Get-Command rg).Source` 解析当前路径
2. 如果解析结果位于 `WindowsApps\\OpenAI.Codex_...`，直接拒绝使用它
3. 再用 `where.exe rg` 找候选路径
4. 只选择非 `WindowsApps\\OpenAI.Codex_` 的独立路径
5. 之后始终通过绝对路径执行 `rg`

例如：

```powershell
& 'C:\Program Files\PowerShell\7\rg.exe' --files
```

## 7. 最后兜底方案

如果当前机器上就是没有可用的独立版 `rg.exe`，或者绝对路径执行仍然失败，那么不要让排查中断。

可以直接退回 PowerShell 原生搜索：

### 7.1 搜文件

```powershell
Get-ChildItem -Recurse -File
```

### 7.2 搜文本

```powershell
Get-ChildItem -Recurse -File | Select-String -Pattern 'keyword'
```

这虽然不如 ripgrep 快，但足够让 AI 继续完成排查和修改。

## 8. 给 AI 的执行原则

如果后续让 AI 在 Windows 上搜索文件或文本，推荐让 AI 按下面这条规则执行：

1. 先解析当前 `rg` 路径
2. 拒绝使用 `WindowsApps\\OpenAI.Codex_...\\resources\\rg.exe`
3. 优先选择机器上独立安装的 `rg.exe`
4. 使用绝对路径执行 `rg`
5. 如果失败，立即退回 PowerShell 原生搜索

这条规则已经很适合沉淀成 skill，而不是每次重新判断。
