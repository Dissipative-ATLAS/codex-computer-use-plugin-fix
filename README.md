# Codex Computer Use plugin unavailable: troubleshooting notes

> English follows Chinese.

## 中文

这是一次真实排障过程的去隐私化总结，针对 Windows 上 Codex Desktop 更新后出现的 **Computer Use 插件不可用** 问题。

这些步骤是临时修复/排障笔记，不是 OpenAI 官方文档。修改 Codex 本地缓存或运行时文件前请保留备份；应用更新后这些改动可能会被覆盖。

### 现象

在 Codex Desktop 的“电脑操控 / Computer Use”设置页看到：

- `Computer Use 插件不可用`
- 或 Codex 无法调用 Computer Use 插件

日志中可能出现类似错误：

```text
Windows Computer Use helper paths are unavailable
bundled_plugins_marketplace_resolve_failed
EBUSY
```

或者：

```text
Package subpath './dist/project/cua/sky_js/src/targets/windows/internal/computer_use_client_base.js'
is not defined by "exports" in ...\@oai\sky\package.json
```

### 根因 1：bundled marketplace 缓存卡住

Codex 会在用户目录下维护 bundled marketplace 缓存。如果更新或后台进程占用导致该目录处于不一致状态，Computer Use、Browser 等 bundled 插件可能显示不可用。

修复方式是关闭相关进程后重命名缓存目录，让 Codex 重新生成。

1. 完全退出 Codex Desktop。
2. 退出 VS Code 中的 Codex/ChatGPT 扩展。
3. 退出相关 Chrome 扩展后端或其他 Codex 后台进程。
4. 打开新的 PowerShell，执行：

```powershell
$target = "$env:USERPROFILE\.codex\.tmp\bundled-marketplaces\openai-bundled"
Rename-Item -LiteralPath $target -NewName "openai-bundled.bak-$(Get-Date -Format yyyyMMdd-HHmmss)"
```

5. 重新打开 Codex Desktop。
6. 回到 Computer Use 设置页检查状态。

如果出现 `访问被拒绝 / Access denied`，通常说明仍有 Codex 相关进程占用该目录。再次确认 Codex、VS Code 扩展、浏览器扩展后台都已退出，或者重启 Windows 后立刻执行上面的重命名命令。

这个操作的风险较低，因为它是重命名缓存目录，不是删除。旧目录会作为 `.bak-时间戳` 备份保留。

### 根因 2：`@oai/sky` 的 `exports` 缺少内部子路径

在某些 Codex/Computer Use 更新组合中，运行时里的 `@oai/sky/package.json` 没有导出 Computer Use 需要的内部文件，导致 Node 抛出 `Package subpath ... is not defined by "exports"`。

可以临时给当前 CUA runtime 的 `@oai/sky/package.json` 增加缺失的 `exports` 项。

先定位最新运行时：

```powershell
$runtime = Get-ChildItem -Directory "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node" |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1

$pkg = Join-Path $runtime.FullName "bin\node_modules\@oai\sky\package.json"
$pkg
```

备份并 patch：

```powershell
$backup = "$pkg.bak-$(Get-Date -Format yyyyMMdd-HHmmss)"
Copy-Item -LiteralPath $pkg -Destination $backup

$json = Get-Content -Raw -LiteralPath $pkg | ConvertFrom-Json

if (-not $json.exports) {
  $json | Add-Member -NotePropertyName exports -NotePropertyValue ([pscustomobject]@{})
}

$indexPath = "./dist/project/cua/sky_js/src/index.js"
$internalPath = "./dist/project/cua/sky_js/src/targets/windows/internal/computer_use_client_base.js"

$json.exports | Add-Member -NotePropertyName "." -NotePropertyValue $indexPath -Force
$json.exports | Add-Member -NotePropertyName $internalPath -NotePropertyValue $internalPath -Force

$json | ConvertTo-Json -Depth 50 | Set-Content -LiteralPath $pkg -Encoding UTF8
```

然后完全退出并重启 Codex Desktop。

### 验证

重启后，在 Codex 里做只读验证即可，不要一上来让插件点击或输入：

```text
请测试 Computer Use 插件是否可用，只列出当前可见应用或窗口，不要点击、输入或切换窗口。
```

当时我的验证结果是插件可以枚举应用和窗口，说明连接正常。

### 回滚

如果修改 `package.json` 后出现新的问题，关闭 Codex 后用备份还原：

```powershell
Copy-Item -LiteralPath "<your-backup-file>" -Destination "<path-to-package.json>" -Force
```

如果重命名了 bundled marketplace 缓存，Codex 重新生成后通常可以保留旧 `.bak-*` 目录一段时间；确认没有问题后再手动删除。

### 注意事项

- 不要公开你的 `auth.json`、日志数据库、订阅链接、API token、账号 ID 或机器名。
- 不要直接删除缓存目录，优先重命名。
- Codex 更新后运行时目录名会变化，所以每次都要重新定位最新的 `cua_node` 目录。
- 这只是针对 Computer Use 插件不可用的本地修复。手机远程控制、浏览器网络、代理/TUN/DNS 问题属于另一类故障。

---

## English

This is a sanitized troubleshooting note from a real Windows Codex Desktop issue where the **Computer Use plugin became unavailable** after an update.

These steps are a workaround and diagnostic record, not official OpenAI documentation. Back up files before changing Codex local cache or runtime files. Updates may overwrite the workaround.

### Symptoms

In Codex Desktop settings, the Computer Use page may show:

- `Computer Use plugin unavailable`
- or Codex cannot invoke the Computer Use plugin

Logs may contain errors like:

```text
Windows Computer Use helper paths are unavailable
bundled_plugins_marketplace_resolve_failed
EBUSY
```

Or:

```text
Package subpath './dist/project/cua/sky_js/src/targets/windows/internal/computer_use_client_base.js'
is not defined by "exports" in ...\@oai\sky\package.json
```

### Cause 1: bundled marketplace cache was stuck

Codex keeps a bundled marketplace cache under the user profile. If an update or a still-running process leaves this directory in a bad state, bundled plugins such as Computer Use or Browser may appear unavailable.

The fix is to close Codex-related processes, rename the cache directory, and let Codex regenerate it.

1. Fully quit Codex Desktop.
2. Quit any Codex/ChatGPT extension running inside VS Code.
3. Quit related Chrome extension backends or other Codex background processes.
4. Open a new PowerShell window and run:

```powershell
$target = "$env:USERPROFILE\.codex\.tmp\bundled-marketplaces\openai-bundled"
Rename-Item -LiteralPath $target -NewName "openai-bundled.bak-$(Get-Date -Format yyyyMMdd-HHmmss)"
```

5. Reopen Codex Desktop.
6. Check the Computer Use settings page again.

If PowerShell returns `Access denied`, a Codex-related process is probably still holding the directory open. Close Codex, VS Code extensions, browser extension backends, or reboot Windows and run the rename command immediately after startup.

This is relatively low risk because it renames the cache instead of deleting it. The old directory remains as a timestamped backup.

### Cause 2: missing `exports` entry in `@oai/sky`

In some Codex/Computer Use update combinations, the `@oai/sky/package.json` file inside the CUA runtime does not export an internal file required by Computer Use. Node then throws `Package subpath ... is not defined by "exports"`.

A temporary workaround is to add the missing export to the current CUA runtime.

Find the newest runtime:

```powershell
$runtime = Get-ChildItem -Directory "$env:LOCALAPPDATA\OpenAI\Codex\runtimes\cua_node" |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1

$pkg = Join-Path $runtime.FullName "bin\node_modules\@oai\sky\package.json"
$pkg
```

Back up and patch:

```powershell
$backup = "$pkg.bak-$(Get-Date -Format yyyyMMdd-HHmmss)"
Copy-Item -LiteralPath $pkg -Destination $backup

$json = Get-Content -Raw -LiteralPath $pkg | ConvertFrom-Json

if (-not $json.exports) {
  $json | Add-Member -NotePropertyName exports -NotePropertyValue ([pscustomobject]@{})
}

$indexPath = "./dist/project/cua/sky_js/src/index.js"
$internalPath = "./dist/project/cua/sky_js/src/targets/windows/internal/computer_use_client_base.js"

$json.exports | Add-Member -NotePropertyName "." -NotePropertyValue $indexPath -Force
$json.exports | Add-Member -NotePropertyName $internalPath -NotePropertyValue $internalPath -Force

$json | ConvertTo-Json -Depth 50 | Set-Content -LiteralPath $pkg -Encoding UTF8
```

Then fully quit and restart Codex Desktop.

### Verification

After restarting, run a read-only check from Codex. Do not start with clicks or typing:

```text
Please test whether the Computer Use plugin is available. Only list visible apps or windows. Do not click, type, or switch windows.
```

In my case, the plugin could enumerate apps and windows after the fix, which confirmed that the connection was healthy.

### Rollback

If editing `package.json` causes problems, close Codex and restore from the backup:

```powershell
Copy-Item -LiteralPath "<your-backup-file>" -Destination "<path-to-package.json>" -Force
```

If you renamed the bundled marketplace cache, keep the `.bak-*` directory for a while. Once Codex works normally, you can delete the old backup manually.

### Notes

- Do not publish your `auth.json`, log databases, subscription URLs, API tokens, account IDs, or machine name.
- Prefer renaming cache directories over deleting them.
- Codex updates can change runtime directory names, so always locate the newest `cua_node` runtime before patching.
- This note is only about the local Computer Use plugin. Phone remote control, browser networking, proxy/TUN, and DNS issues are separate failure classes.
