+++
title = '跨平台密码库同步方案：Password Safe'
date = 2021-04-14T22:02:42+08:00
draft = false
+++

本指南详述如何在 Windows 与 Android 间同步 Password Safe 的 `.psafe3` 数据库文件，使用 InfiniCloud WebDAV 作为云端存储，Rclone 与 FolderSync 作为同步工具。所有操作均已在实际环境中进行过验证。

---

## 1. Password Safe 下载与安装

### Windows 端
- **下载**：访问官方 GitHub Release 页面 [pwsafe/releases](https://github.com/pwsafe/pwsafe/releases)，下载最新版的 `.exe` 或 `.msi` 安装包（例如 `pwsafe-3.71.0.exe`）。
- **安装**：运行安装程序，按向导完成安装，建议保持默认安装路径。
- **创建数据库**：首次打开时选择“新建数据库”，设置**主密码**（务必牢记，无找回手段），并选择存储位置。建议保存在一个本地专用文件夹内，例如 `C:\Sync\PasswordSafe\`。
- **创建 Entry（密码记录）**：
  创建 Entry 即为在库中新建一条密码记录。在软件空白处右键点击并选择 **Add Entry**，在弹出的窗口中依次填写以下字段：
  - `Group`（分组）：选择已有分组，或直接输入新的分组名称进行分类。
  - `Title`（标题）：该条记录的名称，用于在主列表中显示和识别。
  - `Username`（用户名）：登录目标网页或服务时输入的用户名。
  - `Password` 及 `Confirm Password`：登录目标网页时输入的密码。
  - `URL`：目标网页的登录页面网址。
  - `Notes` 编辑框：填写个性化的备注或额外安全提示信息。
  - 填写完毕后，点击 **确定 (OK)** 完成添加。
- **基础使用提醒**：
  - **复制密码**：直接双击某一条目，即可快速将该条目的密码复制到系统剪贴板。
  - **打开网页自动输入用户名密码**：在某一条目上右键点击，选择 **Browse to URL + AutoType**。软件将调用浏览器打开对应的 URL 网址并自动尝试模拟输入账号与密码（注意：由于部分网站前端代码非标准或具有防护机制，模拟输入在少数网页上可能出现兼容性问题）。
  - **手动定位自动输入**：若已手动打开了登录网页，可先清空网页上的用户名密码输入框，并将光标置于用户名输入框中；然后回到软件在对应条目上右键点击，选择 **Perform AutoType**，即可自动填入凭据。
  - **备份**：软件默认会自动生成 `.ibak` 备份文件（与数据库同目录），可结合同步工具一起管理，但需注意过滤规则。
  - **锁文件**：打开数据库时会生成临时锁文件 `.plk`，关闭软件后自动删除。同步时必须排除该文件。

### Android 端
- **下载**：在 Google Play 搜索 “Password Safe”，下载并安装官方安卓版本。
- **初次使用**：打开应用后，需要导入现有的 `.psafe3` 数据库文件。你可以完成后续文件同步配置后，再选择本地的数据库文件。
- **自动填充设置**（红米 K70 未实现）：
  1. 进入 **设置 → 更多设置 → 语言与输入法 → 自动填充服务**。
  2. 如果列表中未出现 Password Safe，可尝试：
     - 重启手机。
     - 进入 **应用管理 → Password Safe**，开启所有相关权限（尤其是“后台弹出界面”和“自启动”）。
     - 在 **设置 → 更多设置 → 无障碍** 中，开启 Password Safe 的辅助功能服务。
     - 清除应用商店缓存后重启手机。
  3. 选择 Password Safe 作为默认自动填充服务。之后登录其他应用或网页时，可通过指纹或主密码验证完成填充。
  4. 注意：安卓版本目前不支持 Windows 端的 “Perform Autotype” 功能，自动填充依赖系统框架或无障碍服务。

---

## 2. InfiniCloud WebDAV 准备
为了保证同步过程的安全，必须使用 InfiniCloud 的**应用密码**而非登录密码，并获取正确的连接地址。
1. 登录 [InfiniCloud My Page](https://infini-cloud.net/en/modules/mypage/usage/)。
2. 进入 **Apps Connection** 区域，找到 **WebDAV Connection**。
3. 记录以下关键信息：
   - **Connection URL**：例如 `https://gima.teracloud.jp/dav/`（请注意以 `/dav/` 结尾）。
   - **Connection ID**：你的用户名，通常是类似 `abc123` 的字符串。
   - **Apps Password**：点击 “Generate” 生成一个专用应用密码。**该密码只显示一次，务必立即复制保存**，后续 Rclone 和 FolderSync 配置都会使用到。
4. 在 InfiniCloud 网页端上创建一个同步文件夹（例如名为 `passwordsafe` ），用于存放 `.psafe3` 文件。

---

## 3. Rclone 安装与配置 (Windows)
Rclone 用于在 Windows 上将本地数据库同步到 InfiniCloud WebDAV，可通过批处理脚本自动运行。

### 安装 Rclone
1. 下载 Windows 版 `rclone.exe`：访问 [Rclone Downloads](https://rclone.org/downloads/)，选择对应的 Intel/AMD 64 位版本。
2. 将 `rclone.exe` 解压到固定目录（例如 `C:\software\rclone\`）。
3. 为方便日常使用，可将该路径添加到系统环境变量 `PATH` 中，或在脚本中直接使用完整路径。

### 配置远程连接
打开命令提示符（CMD）或 PowerShell，执行以下配置命令：

```bash
rclone config
```

依次根据控制台提示进行操作：
1. 输入 `n` 新建远程连接，名称可自定义，例如：`passwordsafe-webdav`。
2. 协议选择 WebDAV，输入：`webdav`。
3. 输入 URL：粘贴 InfiniCloud 的 Connection URL（例如 `https://gima.teracloud.jp/dav/`）。必须保留末尾的 `/`，否则可能导致 400 错误。
4. Vendor 选择：`other`。
5. 输入用户名：填入你的 `Connection ID`。
6. 输入 `y` 设置密码，随后粘贴刚才生成的 `Apps Password`。
7. `bearer_token` 选项：直接回车留空（使用用户名/密码认证）。
8. 确认配置无误，保存并退出。

### 测试连接
运行以下命令列出 WebDAV 根目录，验证连接是否成功：

```bash
rclone lsd passwordsafe-webdav: -vv
```

如需测试列出指定的 `passwordsafe` 文件夹，运行：

```bash
rclone lsd passwordsafe-webdav:passwordsafe -vv
```

*提示：若报错 “400 Bad Request”，请确认 URL 格式，特别是路径是否多写或少写了斜杠。可以通过添加 `--dump bodies` 参数来查看具体的请求包结构进行排查。*

### 同步脚本编写
新建一个文本文件，将其重命名为 `sync_password.bat`，根据实际路径修改并写入以下内容：

```batch
@echo off
set REMOTE_NAME=passwordsafe-webdav
set REMOTE_PATH=passwordsafe
set LOCAL_PATH=C:\Sync\PasswordSafe
set FILTER_RULE=--exclude "*.plk" --exclude "*.ibak"
set LOG_FILE=C:\Sync\sync_log.txt

rclone sync "%LOCAL_PATH%" "%REMOTE_NAME%:%REMOTE_PATH%" %FILTER_RULE% --log-file="%LOG_FILE%" -vv
echo 同步完成。
```

**关键配置说明**：
- `REMOTE_PATH`：为 InfiniCloud 上创建的目标文件夹名称，首位不加斜杠。
- `LOCAL_PATH`：存放本地数据库的路径。
- `FILTER_RULE`：排除了 `.plk` 锁文件与 `.ibak` 自动备份文件，避免多余文件占用空间。
- `--log-file`：用于记录同步日志，便于后续出现异常时排查。测试无误后，可将 `-vv` 调试日志级别改为默认值。

### 手动执行测试
双击运行 `sync_password.bat`，打开日志文件检查是否有错误，观察是否显示 “Transferred” 等传输记录。手动修改数据库后再次执行，确认云端文件已完成对应更新。

---

## 4. Windows 任务计划程序自动化
通过任务计划程序 (`taskschd.msc`)，可以配置定时自动同步任务。

### 创建定时任务（每天固定时间）
1. 打开任务计划程序，点击 **创建基本任务...**，命名为 `PasswordSafe 同步`。
2. 触发器选择 **每天**，设置你习惯的同步时间。
3. 操作选择 **启动程序**：
   - **程序或脚本**：填写 `rclone.exe` 的路径（例如 `C:\software\rclone\rclone.exe`）。
   - **添加参数**：
     ```text
     sync "C:\Sync\PasswordSafe" "passwordsafe-webdav:passwordsafe" --exclude "*.plk" --exclude "*.ibak" --log-file="C:\Sync\sync_log.txt"
     ```
     *（或者直接将“程序或脚本”指向编写好的 `sync_password.bat` 文件，此时参数栏留空）*
4. 保存即可。

---

## 5. FolderSync 安装与配置 (Android)
Android 端使用 FolderSync 应用，将手机本地存储的密码库文件夹与 InfiniCloud 进行同步。

### 添加 InfiniCloud 账户
1. 打开 FolderSync，进入 **Accounts** 菜单，点击右下角 **+** 号新建账户，选择 **WebDAV** 协议。
2. 填写连接参数：
   - **服务器地址**：`gima.teracloud.jp`（提取自 WebDAV 地址的主机名）
   - **端口**：`443`
   - **路径**：`/dav/`
   - **使用 HTTPS**：开启
   - **NTLM 域**：留空
   - **登录名**：`Connection ID`
   - **密码**：`Apps Password`（应用密码）
3. 点击 **测试** 连接，成功后保存。

### 创建同步任务
1. 进入 **Folderpairs**，点击新建同步任务。
2. 配置参数：
   - **账户**：选择刚刚添加的 InfiniCloud 账户。
   - **本地文件夹**：选择手机中存放密码数据库的目录（例如 `/storage/emulated/0/Documents/PasswordSafe`）。
   - **远程文件夹**：浏览并选中云端的 `passwordsafe` 文件夹。
   - **同步类型**：推荐选择 **双向 (Two-way)**，确保任意一端的修改都能在下一次同步中应用。
3. 进入 **高级设置**，启用 **使用计划同步**，并进行如下设置：
   - 选择 **定期同步**，例如同步间隔设为 1 hour（或根据需求调整）。
   - 若支持，可选用“检测到更改时自动同步”，但需注意其可能带来的电池损耗。

### 设置文件过滤
为了避免在手机端同步无用的临时文件和备份：
1. 打开该同步任务配置下的 **Filters**（过滤器）选项。
2. 新增两条规则：
   - 排除文件后缀为 `plk`
   - 排除文件后缀为 `ibak`
   - *（具体语法格式视 FolderSync 版本而定，通常配置排除模式为 `*.plk` 和 `*.ibak`，或在排除列表内添加后缀）*
3. 确认过滤器规则已启用，然后保存该 Folderpair。

---

## 6. Android 端自动填充设置
若安装后在系统的默认自动填充服务列表中无法选定 Password Safe，可依次尝试以下排查步骤：
1. **重启系统**：部分国产 ROM（如澎湃OS、MIUI）在安装新的自动填充服务后，需要重启手机才能刷新服务列表。
2. **权限检查**：进入手机系统的 **应用管理 → Password Safe**，开启“自启动”和“后台弹出界面”权限。
3. **电池策略**：将应用的电池优化策略修改为“无限制”，防止系统在后台终止进程导致无障碍或自动填充服务失效。
4. **无障碍辅助**：若系统原生自动填充框架兼容性不佳，可在“系统设置 → 无障碍”中开启 Password Safe 服务作为备用填充方案。

---

## 7. 常见问题与注意事项
- **主密码安全性**：主密码是加密解密的唯一凭证。若遗忘，数据库将彻底无法打开。请务必将主密码记录在离线介质上安全保存。
- **避免并发编辑**：为规避多设备同步时的覆盖冲突，建议避免在两台设备上同时打开并修改 `.psafe3` 文件。在其中一台编辑并同步完毕后，再在另一台更新并打开。
- **锁文件 `.plk` 的作用**：当数据库处于打开状态时，本地会生成此锁定文件以防二次写入。如通过云端同步了该锁文件，会导致另一台设备打开时误报“只读锁”，因此在同步工具中**必须排除该文件**。
- **冲突处理机制**：一旦因网络或操作不当引发同步冲突，本地会产生诸如 `original (conflicted copy...).psafe3` 的冲突副本。此时可利用电脑端 Password Safe 的菜单功能 **File → Merge**，将副本中的新修改合并到主数据库中，合并完毕后再删除冲突文件。
- **单向同步与双向同步选择**：在 Rclone 中使用的 `sync` 默认是单向强制同步（本地覆盖远程），适合把 Windows 端作为主要编辑源的场景。如需完整的双向同步，请在 Rclone 中改用 `bisync` 指令（初次使用请务必备份数据），或使用支持双向同步的客户端。FolderSync 端直接开启“双向”即可。
- **网络连接失败（400 错误）**：若 Rclone 测试返回 400 Bad Request，多数是由于配置里的 WebDAV 路径前缀格式不规范引起。检查并确认你的 URL 路径是否与 InfiniCloud 官方提供的路径（通常为 `/dav/`）一致。
