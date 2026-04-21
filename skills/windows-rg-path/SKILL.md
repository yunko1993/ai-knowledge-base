---
name: windows-rg-path
description: >-
  Use when working in Windows terminals where `rg` may resolve to an inaccessible bundled binary.
  Apply whenever searching files or text with ripgrep (`rg`, `rg --files`, `rg -n`) to force
  resolution of the machine-local rg.exe path and execute ripgrep by absolute path.
---

# Windows RG Path

目标：在 Windows 下优先使用机器上独立可访问的 `rg.exe`，避免误用 Codex 内置的 `WindowsApps` 路径。

## 核心规则

1. 在当前任务第一次调用 ripgrep 前，先解析路径：

   ```powershell
   (Get-Command rg).Source
   ```

2. 如果解析结果指向：

   ```text
   C:\Program Files\WindowsApps\OpenAI.Codex_...\app\resources\rg.exe
   ```

   就把它视为 Codex 内置二进制，不要作为首选 `rg`。

3. 如果路径为空、命中内置路径、或调用失败，就执行：

   ```powershell
   where.exe rg
   ```

   选择第一个不在 `WindowsApps\OpenAI.Codex_` 下的有效 `.exe` 路径。

4. 一旦找到独立版 `rg.exe`，后续调用都使用绝对路径执行，例如：

   ```powershell
   & 'C:\Program Files\PowerShell\7\rg.exe' --files
   ```

   ```powershell
   & 'C:\Program Files\PowerShell\7\rg.exe' 'pattern' -n .
   ```

5. 如果绝对路径执行报错（例如 access denied），立即退回 PowerShell 原生搜索，不要卡住任务。

6. 不要假设 alias 或 shell 状态能跨工具调用保留；每次都按“新 shell”处理。

## 兜底策略

如果找不到可用的独立版 `rg.exe`，或者绝对路径执行失败，就直接改用 PowerShell 原生搜索：

### 搜文件

```powershell
Get-ChildItem -Recurse -File
```

### 搜文本

```powershell
Get-ChildItem -Recurse -File | Select-String -Pattern 'keyword'
```

## 快速执行顺序

1. 解析 `rg` 路径
2. 拒绝 `WindowsApps\OpenAI.Codex_...\resources\rg.exe`
3. 选择机器上独立安装的 `rg.exe`
4. 用绝对路径执行 ripgrep
5. 失败就退回 PowerShell 原生搜索
