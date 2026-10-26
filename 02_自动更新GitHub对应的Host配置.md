# 自动更新GitHub对应的Host配置


针对 GitHub 时断时续的问题，单纯复制粘贴网上的 IP 列表往往过几天就会失效，因为 GitHub 的 IP 会动态变化。比较理想的“神器”思路有两种：

- 一是用脚本定时从维护良好的数据源同步（简单、可控）；

- 二是用工具在本地实时测速优选 IP（更精准、能自愈）

---

## 1. 方案一：手写 PowerShell 脚本（轻量、可定制）
这个方案的核心逻辑是：从目前维护最稳定的 GitHub520 项目拉取最新的 hosts 列表，用标记块的方式写入系统 hosts 文件（避免覆盖你已有的自定义内容），然后自动刷新 DNS。GitHub520 通过 GitHub Action 定时更新 IP，数据源地址无需访问 GitHub 即可获取。

### 1) 脚本代码

将以下代码保存为 Update-GitHubHosts.ps1：
```bash
# Update-GitHubHosts.ps1
# 必须以管理员身份运行

$HostsPath = "$env:SystemRoot\System32\drivers\etc\hosts"
$SourceUrl = "https://raw.hellogithub.com/hosts"
$StartMarker = "# GitHub Host Start"
$EndMarker = "# GitHub Host End"

# 1. 检查管理员权限
$isAdmin = ([Security.Principal.WindowsPrincipal] [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
if (-not $isAdmin) {
    Write-Error "请以管理员身份运行此脚本（右键 -> 以管理员身份运行）。"
    exit 1
}

# 2. 下载最新 hosts 内容
Write-Host "正在从 GitHub520 获取最新 hosts..."
try {
    $response = Invoke-WebRequest -Uri $SourceUrl -UseBasicParsing -TimeoutSec 15
    $newBlock = $response.Content.Trim()
} catch {
    Write-Error "下载失败，请检查网络连接：$($_.Exception.Message)"
    exit 1
}

if ([string]::IsNullOrWhiteSpace($newBlock)) {
    Write-Error "下载内容为空，中止更新。"
    exit 1
}

# 3. 读取当前 hosts 内容
$currentContent = Get-Content -Path $HostsPath -Raw -Encoding UTF8
if (-not $currentContent) { $currentContent = "" }

# 4. 构造新的完整 hosts 内容（原子替换标记块）
$blockPattern = "(?s)$([regex]::Escape($StartMarker)).*?$([regex]::Escape($EndMarker))"
$replacement = "$StartMarker`r`n$newBlock`r`n$EndMarker"

if ($currentContent -match $blockPattern) {
    # 如果已存在标记块，只替换块内内容
    $updatedContent = $currentContent -replace $blockPattern, $replacement
    Write-Host "检测到已有 GitHub 标记块，正在执行替换..."
} else {
    # 如果没有标记块，追加到文件末尾
    $updatedContent = $currentContent.TrimEnd() + "`r`n`r`n$replacement`r`n"
    Write-Host "未找到标记块，正在追加新的 GitHub hosts 段..."
}

# 5. 写回文件（使用 UTF8 无 BOM，避免 Windows 解析异常）
[System.IO.File]::WriteAllText($HostsPath, $updatedContent, (New-Object System.Text.UTF8Encoding $false))
Write-Host "hosts 文件已更新。"

# 6. 刷新 DNS 缓存
Write-Host "正在刷新 DNS 缓存..."
ipconfig /flushdns | Out-Null
Write-Host "完成。GitHub 的 hosts 记录已更新并生效。"
```

### 2) 如何使用

- 手动运行：右键点击 `Update-GitHubHosts.ps1`，选择“以管理员身份运行”。如果遇到执行策略限制，可以在管理员 PowerShell 中先执行 Set-ExecutionPolicy -Scope Process Bypass，再运行脚本。

- 设置为计划任务（自动更新） ：打开“任务计划程序”，创建基本任务，触发器设为“每天”或“登录时”，操作设为“启动程序”，程序填 powershell.exe，参数填 `-ExecutionPolicy Bypass -File "你的脚本完整路径\Update-GitHubHosts.ps1"`，并勾选“使用最高权限运行”

### 3) 优缺点

- **优点**：代码透明，完全可控；只修改标记块内的内容，不影响你 hosts 文件里的其他条目；数据源稳定。

- **局限**：本质上仍然是“拉取一份别人维护的静态列表”，如果 GitHub520 的 IP 在你当前网络环境下恰好质量不佳，脚本无法自动感知和切换。

---

## 2. 方案二：直接使用 ghprobe（“自愈型”神器）
如果你不想自己维护脚本，或者对“静态列表”的可靠性心存疑虑，ghprobe 是更接近“神器”定位的工具。它专门为 Windows 设计，核心思路是不依赖任何第三方静态列表，而是在你自己的网络里实时探测。

它的工作流程是：本地 DoH 解析（绕开可能被污染的 DNS）→ 对每个候选 IP 依次进行 TCP 延迟、TLS 证书校验、HTTP 应用层三关探测 → 按延迟排名，写入最优 IP；随后通过 Windows 计划任务每分钟做一次健康检查，一旦当前 IP 失效，自动重新探测并替换。全程只修改 hosts 文件，不安装代理、驱动或服务。

安装与使用（需要 Node.js ≥ 18）：
```bash
npm install -g ghprobe
ghprobe install
```
`ghprobe install` 会完成一次探测、写入 hosts，并注册计划任务，全程只弹一次 UAC 提权。之后你不需要再手动干预。卸载时执行` ghprobe uninstall`，会删除计划任务并还原 hosts。

- 优点：IP 由本地实测选出，适配你的实际网络；具备“自愈”能力，IP 失效后自动替换，无需人工介入；只改 hosts，干净无残留。
- 局限：仅支持 Windows，且需要 Node.js 环境。

---

## 3. 方案三：不想碰命令行？SwitchHosts 是成熟选择

如果你更倾向于图形界面，SwitchHosts 是一款开源的 hosts 管理工具，支持添加 Remote 类型的配置：填入 GitHub520 的地址 `https://raw.hellogithub.com/hosts`，
将自动刷新设为 1 小时，它就会定时拉取并应用更新，需要时会提示提权。它适合希望“零代码”管理 hosts 的用户，但同样属于订阅静态列表的模式，不具备 ghprobe 那种实时测速与自愈能力。


