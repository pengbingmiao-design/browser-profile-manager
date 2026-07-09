# 浏览器多账号管理器

## 项目概述
tkinter GUI 工具，为每个"用户"启动独立的浏览器实例（Edge/Chrome），
每个实例使用独立的 `--user-data-dir`，实现 Cookie/Session 隔离。

## 核心文件
| 文件 | 作用 |
|------|------|
| main.py | 主程序，含 UI + 数据管理 + 激活弹窗 + 更新弹窗 |
| license.py | 激活模块：机器码生成、HMAC-SHA256(含机器码)签名验证、XOR混淆存储 |
| update.py | 更新模块：GitHub 版本检测、下载、自替换重启 |
| keygen.py | 卖家工具：`--file --mid <机器码>` 或交互式生成绑定机器的 license.key |
| version.json | 版本信息模板（提交到 GitHub 仓库） |
| 生成激活码.bat | 一键启动 keygen 交互模式 |
| run.bat | 开发测试用启动脚本 |

## 架构
- UI：纯 tkinter（无 ttk 主题依赖），PanedWindow 左右分栏
- 数据：JSON 存储在 `~/.browser_profile_manager/config.json`
- 用户数据：`~/.browser_profile_manager/users/<uid>/`
- 激活状态：`~/.browser_profile_manager/license.dat`（XOR混淆）

## 激活系统（v2 — 机器码绑定）

### 激活码格式
激活码(32位) = 随机16位 + HMAC-SHA256(随机16位 + 目标机器码, 内置密钥)[:16]
激活码在生成时即绑定目标机器码，换电脑签名验证直接失败。

### 机器码
SHA256(MAC地址 + 计算机名 + 用户名)[:16] → 固定16位十六进制，除非换网卡/重装系统永久不变。

### 客户侧流程
双击 exe → 未激活 → 弹窗显示机器码 → 客户复制发给卖家
→ 卖家生成 license.key → 客户放入 exe 同目录
→ 双击 exe → 自动激活 → license.key 被删除 → 进入主界面

### 卖家侧流程
双击 生成激活码.bat → 粘贴客户机器码 → 回车 → 生成 license.key
或：python keygen.py --file --mid <机器码>

## 自动更新系统

启动时通过 raw.githubusercontent.com 读取 version.json，版本号高于当前则弹窗。

- 立即更新 → 下载新 exe（自动 P2P 校验）→ 批处理替换（最多 5 次重试）→ 自动重启
- 稍后提醒 → 正常启动，下次再提示
- 网络异常 → 静默跳过，不影响使用
- GitHub 仓库需为公开（无需 token）

### 下载镜像
国内直连 GitHub 可能失败，代码内置镜像自动 fallback：
```python
_MIRRORS = [
    "",                       # 0. 直连 GitHub
    "https://gh.ddlc.top/",   # 1. ddlc 加速
    "https://ghproxy.net/",   # 2. ghproxy 加速
]
```
新增镜像需验证：`curl -sI "镜像/https://raw.githubusercontent.com/..."` 返回 200 才算可用。

### 下载安全校验
- 下载后校验 PE 文件头（`MZ`），非真 exe 自动丢弃
- 文件大小与 Content-Length 不匹配则丢弃
- 防止镜像返回 HTML 错误页被误装

### 已知问题：PyInstaller DLL 加载失败
杀毒软件可能拦截 PyInstaller 从 `%TEMP%\_MEI*` 加载 `python310.dll`。
症状：`LoadLibrary: 找不到指定的模块`。
解决：将 exe 所在目录加入杀软白名单。

### 发布新版本流程

```
你只需要修改两个文件：
  1. update.py      → APP_VERSION = "2.5.0"
  2. version.json   → version + url + notes 全部更新

然后告诉 Claude：
  "打包并发布 v2.5.0"
```

Claude 会自动完成：
1. 确认两个文件版本号一致
2. PyInstaller 打包 → `dist\BrowserProfileManager_v2.5.0.exe`
3. Git commit & push（只推 version.json，源码保密）
4. `gh release create v2.5.0` → 上传 exe
5. 所有客户下次启动自动收到更新提示


### ⚠️ 文件名一致性（出错会导致所有客户 404）

上传到 GitHub Release 的文件名和 version.json 里的 URL 文件名**必须完全一致**，每个版本都用带 v 后缀的独立文件名：

```
version.json → "url": ".../BrowserProfileManager_v2.5.0.exe"
gh upload    → BrowserProfileManager_v2.5.0.exe   ← 一个字不能差
```

### 测试更新系统
```
1. 修改 update.py → APP_VERSION = "2.4.1"（旧版本）
2. 重新打包生成旧版 exe
3. 确保 GitHub 上 version.json 是更高版本（如 2.4.2）
4. 运行旧版 exe → 应弹出更新提示
```

## 入口流程
main() → root.withdraw() → is_activated()?
  ├─ 否 → _try_auto_activate() → 失败则 ActivationDialog
  └─ 是 → root.deiconify() → 更新检查 → 正常启动

## 打包

- Python: 3.10 (`C:\Users\Novice_Anony\AppData\Local\Programs\Python\Python310\python.exe`)
- PyInstaller: 6.21.0
- 命令：`pyinstaller --onefile --noconsole --hidden-import license --hidden-import update --icon=icon.ico --add-data "icon.ico;." --name BrowserProfileManager main.py`
- 输出：`dist\BrowserProfileManager.exe` (~9MB)

### 打包后本地版本标注
每次打包后，将 exe 重命名带版本号，方便区分：
```
dist\BrowserProfileManager_v2.4.2.exe
```
同时保留一份无版本号的 `BrowserProfileManager.exe` 用于测试。

### 涉及版本号的文件（发版时需全部修改）
| 文件 | 字段 | 说明 |
|------|------|------|
| `update.py:35` | `APP_VERSION = "x.y.z"` | 内置版本号，打包进 exe |
| `version.json` | `"version"` + `"url"` + `"notes"` | 推送到 GitHub，客户端据此判断更新 |

两个文件的版本号必须一致（打包时的 update.py 版本 = version.json 版本）。

## 配色
C_PRIMARY=#1A73E8, C_DANGER=#D93025, C_BG=#EEF1F5, C_CARD=#FFFFFF
