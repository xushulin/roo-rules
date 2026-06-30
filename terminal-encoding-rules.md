## 终端编码规范（全局）

你在通过 [`execute_command`]() 执行 CLI 命令时，**必须**遵守以下终端编码规范，无需再次请求批准。本规则适用于所有项目、所有工作区。

### 一、背景

Windows 系统下，默认 Shell（cmd.exe）和 PowerShell 的终端输出编码通常是 **GBK（CP936）**，而非 UTF-8。当命令输出包含中文字符时，若终端编码与工具预期不一致，输出中的中文会变成乱码，Agent 无法正确读取命令结果。

Linux/macOS 系统默认使用 UTF-8，通常不存在此问题。

### 二、cmd.exe 编码配置（强制）

项目默认 Shell 为 cmd.exe 时，**任何可能包含中文输出的命令**，都必须在命令前添加 `chcp 65001 > nul &&`：

```
chcp 65001 > nul && (你的原始命令)
```

| 元素 | 说明 |
|------|------|
| `chcp 65001` | 将当前控制台代码页切换为 UTF-8 |
| `> nul` | 抑制切换成功的提示信息（`Active code page: 65001`），避免污染命令输出 |
| `&&` | 确保切换成功后才执行后续命令 |

**适用范围**：任何可能产生中文输出的命令，包括但不限于：

| 场景 | 示例 |
|------|------|
| Python 脚本输出中文 | `python train.py`（日志含中文） |
| Git 操作 | `git log`、`git diff`（commit 信息含中文） |
| 文件内容查看 | `type file.txt`（文件内容含中文） |
| 编译构建 | `npm run build`（构建日志含中文） |

**不需要 `chcp 65001` 的场景**（纯 ASCII 输出，跳过以避免冗余）：

- `git status`（文件路径不含中文时）
- 纯英文 API 调用
- 数值计算类命令

### 三、PowerShell 编码配置（强制）

通过 [`execute_command`]() 调用 PowerShell 时，**必须**在命令开头设置输出编码为 UTF-8：

```
powershell -NoProfile -Command "[Console]::OutputEncoding = [Text.Encoding]::UTF8; (你的原始命令)"
```

| 元素 | 说明 |
|------|------|
| `-NoProfile` | 跳过 profile 加载，避免干扰 |
| `[Console]::OutputEncoding = [Text.Encoding]::UTF8` | 设置终端输出编码为 UTF-8 |

### 四、输出重定向的编码影响

终端编码不仅影响屏幕输出，还影响通过管道和重定向写入文件时的编码：

| Shell | 重定向方式 | 默认编码 | 后果 |
|-------|-----------|---------|------|
| cmd.exe | `>` | 保持程序输出的原始字节 | 配合 `chcp 65001` 后程序输出为 UTF-8，重定向保留 UTF-8 字节，**安全** |
| PowerShell | `>` | **UTF-16 LE**（`Out-File` 默认） | 直接使用时目标文件为 UTF-16 LE，后续以 UTF-8 读取会乱码 |
| PowerShell | `\| Out-File -Encoding UTF8` | UTF-8 | ✅ 推荐 |

**PowerShell 中需要将输出写入文件时，必须使用 `| Out-File -Encoding UTF8` 替代裸 `>`。**

### 五、自检清单

每次通过 [`execute_command`]() 执行命令前，自问：

- [ ] 当前默认 Shell 是 cmd.exe 还是 PowerShell？
- [ ] 命令输出中是否可能包含中文？
- [ ] 若为 cmd.exe，是否已添加 `chcp 65001 > nul &&`？
- [ ] 若为 PowerShell，是否已设置 `[Console]::OutputEncoding = [Text.Encoding]::UTF8;`？
- [ ] 若有输出重定向，是否确保了目标文件编码为 UTF-8？

### 六、禁止行为

- ❌ 在 cmd.exe 中执行含中文输出的命令时不设置 `chcp 65001`
- ❌ 在 PowerShell 中使用裸 `>` 将含中文的输出写入文件
- ❌ 假设 Windows 终端默认编码为 UTF-8
- ❌ 遇到中文乱码后不修复编码设置而直接忽略
- ❌ 在 `execute_command` 中连续执行多条命令时只设置第一条的编码
