## 文件编码与换行符规则（全局）

你在创建或修改任何文本文件时，**必须**遵守以下编码和换行符规范，无需再次请求批准。本规则适用于所有项目、所有工作区、所有操作模式（Code、Debug、Architect 等）。

### 一、强制编码规范

1. **所有文本文件必须使用 UTF-8 编码（无 BOM）**，但 `.md` 文件例外：`.md` 文件允许带 UTF-8 BOM（`EF BB BF`），因为部分工具在 GBK 环境中依赖 BOM 识别编码。
2. **禁止**使用 GBK、GB2312、Latin-1 等非 UTF-8 编码写入任何新文件或修改现有文件。
3. **禁止**使用 `Out-File -Encoding default` 等依赖系统默认编码的方式写入文件。Windows 系统默认编码为 GBK，会破坏 UTF-8 内容。

### 二、写入方式约束

| 场景 | 正确做法 | 禁止做法 |
|------|---------|---------|
| PowerShell 写文本 | `[System.IO.File]::WriteAllText($path, $content, [System.Text.Encoding]::UTF8)` | `Out-File -Encoding default`、`Set-Content`（默认 ASCII） |
| PowerShell 重定向输出 | `\| Out-File -Encoding UTF8` | 裸 `>`（默认编码为 UTF-16 LE） |
| cmd.exe 重定向输出 | 先执行 `chcp 65001` 将控制台代码页切换为 UTF-8，再用 `>` 重定向 | 在 GBK 代码页下将含中文的输出重定向到文件 |
| Git 提取历史文件 | `git cat-file blob <rev>:<path> > <dest>` | `git show <rev>:<path> \| Out-File`（管道会破坏二进制字节） |
| 各语言代码写文件 | 明确指定 UTF-8（Kotlin/Java 用 `Charsets.UTF_8`，Python 用 `encoding='utf-8'`） | 依赖平台默认 charset |

### 三、外部数据兼容（读取非 UTF-8 文件）

若需要读取可能为其他编码的外部文件（如 GBK 编码的 CSV/日志），读取时**必须显式指定正确的源编码**（如 `encoding='gbk'`），转换后再以 UTF-8 写入目标文件。**禁止**用默认编码盲读后直接转存，否则会产生乱码。

### 四、Git 操作注意事项

1. **二进制文件内容传递**：从 Git 提取历史文件内容时，**必须用 `git cat-file blob` 直接重定向到文件**，**禁止**通过管道 `|` 将内容传递给其他命令后再写入文件。管道会将字节流当作文本处理，在 Windows PowerShell 中可能转换编码或破坏换行符。
2. **`git show` / `git diff`** 的输出默认经过文本编码转换，不适用于提取原始字节，仅用于人工查看差异。

### 五、修改文件后的自检清单

每次修改 `.md`、`.kt`、`.java`、`.py`、`.xml`、`.gradle.kts`、`.toml`、`.properties`、`.json`、`.yaml` 等文本文件后，自问：

- [ ] 写入时是否明确指定了 UTF-8 编码？
- [ ] 是否避免了 `Out-File -Encoding default` 等依赖系统编码的命令？
- [ ] 如果是通过 PowerShell 操作，是否使用了 `[System.IO.File]::WriteAllText()` 或 `-Encoding UTF8` 参数？
- [ ] 读取外部文件时，是否显式指定了正确的源编码？
- [ ] 文件是否仍是多行格式（换行符未丢失）？
- [ ] 如果文件含中文，用编辑器打开确认内容可读？

### 六、诊断方法

如怀疑某文件编码异常，可用以下任一方式核验：

- 用支持显示编码的编辑器（如 VS Code 右下角状态栏）打开，确认显示为 `UTF-8`。
- 用 PowerShell 读取文件头字节，判断是否含 BOM、是否存在乱码字符。
- 含中文的文件直接肉眼检查是否出现"锟斤拷""�"等典型乱码。

### 七、禁止行为

- ❌ 使用默认编码写入任何文本文件
- ❌ 通过管道传递 Git 对象内容后再写文件
- ❌ 在写文件工具中使用非 UTF-8 内容
- ❌ 读取外部非 UTF-8 文件时不指定源编码
- ❌ 修改文件后不检查换行符是否正常

### 八、例外说明

若因特殊原因必须使用非 UTF-8 编码（如对接遗留系统的固定编码要求），必须在操作前向用户明确说明并获得确认。
