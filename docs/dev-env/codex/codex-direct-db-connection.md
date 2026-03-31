# 在 Codex 中让模型直接连接本机数据库（Windows / MySQL）

## 1. 目标（面向 AI 自治）

这篇文档的目标不是“告诉人怎么点界面”，而是让 AI 在拿到文档后可自行完成：

1. 判断 `mysql` 是否可用
2. 自动定位 `mysql.exe`
3. 自动把 MySQL `bin` 目录写入用户 `PATH`
4. 验证连接并执行只读 SQL

这里是“本机代执行”，不是云端直连内网数据库。

## 2. 使用前提

1. 当前机器本身可访问目标数据库。
2. 你提供连接参数：`host` / `port` / `database` / `user` / `password`。
3. 默认只读，不执行写操作。

## 3. AI 标准排查流程（必须按顺序）

### 步骤 1：检查 `mysql` 是否已在 PATH

```powershell
mysql --version
```

如果成功，直接跳到“步骤 4”。

### 步骤 2：自动查找 `mysql.exe`

先查常见安装路径：

```powershell
$candidates = @(
  "C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe",
  "C:\Program Files\MySQL\MySQL Server 5.7\bin\mysql.exe",
  "C:\Program Files\MariaDB 10.11\bin\mysql.exe"
)
$mysqlExe = $candidates | Where-Object { Test-Path $_ } | Select-Object -First 1
$mysqlExe
```

如果还没找到，再全盘定向搜索：

```powershell
$mysqlExe = Get-ChildItem C:\ -Recurse -Filter mysql.exe -ErrorAction SilentlyContinue |
  Where-Object { $_.FullName -match "\\bin\\mysql\.exe$" } |
  Select-Object -ExpandProperty FullName -First 1
$mysqlExe
```

### 步骤 3：把 `mysql.exe` 所在目录写入用户 PATH

```powershell
$mysqlBin = Split-Path $mysqlExe -Parent
$userPath = [Environment]::GetEnvironmentVariable("Path", "User")
if (-not (($userPath -split ';') -contains $mysqlBin)) {
  $newUserPath = if ([string]::IsNullOrWhiteSpace($userPath)) { $mysqlBin } else { "$userPath;$mysqlBin" }
  [Environment]::SetEnvironmentVariable("Path", $newUserPath, "User")
}

# 让当前会话立即生效
if (-not (($env:Path -split ';') -contains $mysqlBin)) {
  $env:Path = "$env:Path;$mysqlBin"
}

mysql --version
```

### 步骤 4：验证数据库连通性

```powershell
mysql -h <host> -P <port> -u <user> -p -D <database> -e "SELECT 1;"
```

出现 `1` 代表基础链路可用。

### 步骤 5：执行只读 SQL

```sql
SELECT DATABASE() AS db, NOW() AS now_time;
SHOW TABLES LIKE 'your_table_name';
SELECT * FROM your_table_name LIMIT 20;
SELECT COUNT(*) AS total FROM your_table_name;
```

## 4. 给 Codex 的可复制提示词

```text
按这篇文档执行数据库接入排查，必须严格按顺序：
1) 先检查 mysql --version；
2) 若失败，自动查找 mysql.exe；
3) 将 mysql.exe 所在 bin 目录写入用户 PATH，并让当前会话立即生效；
4) 再次验证 mysql --version；
5) 执行 SELECT 1 验证连通性；
6) 执行我给的只读 SQL。
每一步都输出：命令、结果、结论、下一步。
禁止未经确认执行 UPDATE/DELETE/TRUNCATE/DDL。
```

## 5. 常见失败与处理

### 5.1 找不到 `mysql.exe`

1. 扩展搜索范围到其他盘符（如 `D:\`）。
2. 确认是否仅安装了 MySQL Server 但未安装客户端组件。

### 5.2 PATH 写入后仍不生效

1. 当前会话需要补 `env:Path`（文档步骤 3 已包含）。
2. 新开终端后再次执行 `mysql --version`。

### 5.3 连接失败（超时/拒绝/认证失败）

1. 检查网络与白名单。
2. 检查账号权限与库名。
3. 检查是否连错环境。

## 6. 安全基线

1. 默认只读。
2. 写操作必须二次确认。
3. 先测测试库，再碰生产库。
