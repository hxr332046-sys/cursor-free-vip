# Cursor Free VIP - 技术架构文档

## 目录
1. [项目概述](#1-项目概述)
2. [系统架构](#2-系统架构)
3. [核心模块详解](#3-核心模块详解)
4. [技术实现细节](#4-技术实现细节)
5. [数据流分析](#5-数据流分析)
6. [跨平台实现](#6-跨平台实现)
7. [安全与防护机制](#7-安全与防护机制)
8. [构建与部署](#8-构建与部署)
9. [关键技术要点总结](#9-关键技术要点总结)

---

## 1. 项目概述

### 1.1 项目定位
Cursor Free VIP 是一个用于 Cursor AI 编辑器的自动化工具，主要功能包括：
- **OAuth 自动注册**: 支持 Google/GitHub 账号自动注册（DrissionPage 驱动）
- **机器码重置**: 修改本地与系统级标识符，并可 Patch `main.js` / `workbench.desktop.main.js`
- **账号与维护**: 手动/自动注册、临时邮箱、删除 Google 绑定、用量展示、退出进程、禁用自动更新、绕过版本检查等（见 `main.py` 菜单）

### 1.2 技术栈

| 层级 | 技术 | 用途 |
|------|------|------|
| **核心语言** | Python 3.x | 主要开发语言 |
| **浏览器自动化** | DrissionPage（主路径）+ Selenium（辅助） | Google/GitHub OAuth 以 DrissionPage 为主；`github_cursor_register.py` 等使用 Selenium |
| **数据存储** | SQLite + JSON | 本地配置和状态管理 |
| **UI 渲染** | Colorama | 终端彩色输出 |
| **打包工具** | PyInstaller | 生成可执行文件 |

### 1.3 项目结构

```
cursor-free-vip/
├── main.py                      # 程序入口、Translator、菜单与版本检查
├── logo.py                      # Logo 与版本号展示
├── config.py                    # 配置管理（默认项 + 各平台路径）
├── utils.py                     # 路径、Chrome 探测、随机等待 get_random_wait_time 等
├── cursor_auth.py               # Cursor 本地认证写入 state.vscdb
├── oauth_auth.py                # OAuthHandler：DrissionPage + Google/GitHub 流程
├── cursor_register_google.py    # Google 注册入口（调用 OAuth）
├── cursor_register_github.py    # GitHub 注册入口（调用 OAuth）
├── cursor_register.py           # 自动注册（CursorRegistration）
├── cursor_register_manual.py    # 手动注册流程
├── new_signup.py / new_tempemail.py   # 注册与临时邮箱辅助
├── github_cursor_register.py    # 基于 Selenium 的 GitHub 相关注册（菜单「临时 GitHub」）
├── delete_cursor_google.py      # CursorGoogleAccountDeleter：删除 Google 绑定
├── reset_machine_manual.py      # MachineIDResetter：机器码重置（菜单常用）
├── totally_reset_cursor.py      # 另一套 MachineIDResetter + 完全重置逻辑（菜单「完全重置」）
├── cursor_acc_info.py           # 账号用量/信息展示（Utils.enabled_account_info）
├── quit_cursor.py               # 结束 Cursor 进程
├── disable_auto_update.py       # 禁用 Cursor 自动更新
├── bypass_version.py            # 绕过版本检查
├── build.py / build.spec        # 构建脚本与 PyInstaller 规格
├── block_domain.txt             # 域名规则（随 spec 打包）
├── locales/                     # 多语言 JSON（约 12 种）
├── scripts/                     # 安装脚本 install.ps1 / install.sh
├── turnstilePatch/              # 扩展：Turnstile / 验证相关
└── PBlock/                      # 扩展：请求拦截与规则（与 turnstilePatch 一并打包）
```

---

## 2. 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        用户界面层                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ 主菜单/账号   │  │ 语言/Chrome   │  │ 配置/禁用更新/    │   │
│  │ 信息展示      │  │ Profile 选择  │  │ 绕过版本/贡献链接 │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        业务逻辑层                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ OAuth / 手动  │  │ 机器码重置    │  │ 删除 Google 账号  │   │
│  │ / 临时 GitHub │  │ / 完全重置    │  │ / 用量查询        │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        数据操作层                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ SQLite 操作  │  │  JSON 配置   │  │  文件系统操作     │   │
│  │              │  │              │  │                  │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                        系统交互层                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │ 浏览器控制   │  │ 注册表操作   │  │  进程管理        │   │
│  │ (Chrome)    │  │ (Windows)   │  │                  │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 核心类设计

```
┌──────────────────────────────────────────────────────────────┐
│                        OAuthHandler                           │
├──────────────────────────────────────────────────────────────┤
│  职责: 处理 OAuth 认证流程 (Google/GitHub)                     │
├──────────────────────────────────────────────────────────────┤
│  方法:                                                        │
│  - setup_browser()           # 初始化浏览器                   │
│  - handle_google_auth()      # Google 认证                   │
│  - handle_github_auth()      # GitHub 认证                   │
│  - _wait_for_auth()          # 等待认证完成                  │
│  - _extract_auth_info()      # 提取认证信息                  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                        CursorAuth                             │
├──────────────────────────────────────────────────────────────┤
│  职责: 管理 Cursor 本地认证数据                               │
├──────────────────────────────────────────────────────────────┤
│  方法:                                                        │
│  - update_auth()             # 更新认证信息                  │
│  - 操作 SQLite: state.vscdb                                 │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                     MachineIDResetter                         │
├──────────────────────────────────────────────────────────────┤
│  职责: 重置机器标识符                                         │
├──────────────────────────────────────────────────────────────┤
│  方法:                                                        │
│  - generate_new_ids()        # 生成新 ID                     │
│  - update_sqlite_db()        # 更新 SQLite                   │
│  - update_system_ids()       # 更新系统 ID                   │
│  - reset_machine_ids()       # 执行重置                      │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                        Translator                             │
├──────────────────────────────────────────────────────────────┤
│  职责: 多语言国际化支持                                       │
├──────────────────────────────────────────────────────────────┤
│  方法:                                                        │
│  - detect_system_language()  # 检测系统语言                  │
│  - load_translations()       # 加载语言包                    │
│  - get()                     # 获取翻译文本                  │
└──────────────────────────────────────────────────────────────┘
```

**其他重要类（分散在各模块）**

| 类名 | 文件 | 说明 |
|------|------|------|
| `CursorGoogleAccountDeleter` | `delete_cursor_google.py` | 继承 `OAuthHandler`，删除当前 Google 绑定 |
| `GitHubCursorRegistration` | `github_cursor_register.py` | Selenium 驱动的 GitHub 注册流程 |
| `CursorRegistration` | `cursor_register.py` / `cursor_register_manual.py` | 自动 / 手动注册封装 |
| `UsageManager` / `Config` | `cursor_acc_info.py` | 读取并展示账号用量等信息 |
| `AutoUpdateDisabler` | `disable_auto_update.py` | 关闭 Cursor 自动更新 |
| `CursorQuitter` | `quit_cursor.py` | 退出 Cursor 进程 |
| `NewTempEmail` | `new_tempemail.py` | 临时邮箱辅助 |

---

## 3. 核心模块详解

### 3.1 OAuth 认证模块 (oauth_auth.py)

#### 3.1.1 认证流程

```
┌─────────┐     ┌──────────────┐     ┌─────────────────┐     ┌──────────┐
│  启动   │────▶│ 初始化浏览器  │────▶│ 访问注册页面     │────▶│点击OAuth │
└─────────┘     └──────────────┘     └─────────────────┘     └────┬─────┘
                                                                   │
                                                                   ▼
┌─────────┐     ┌──────────────┐     ┌─────────────────┐     ┌──────────┐
│ 完成    │◀────│ 保存Token    │◀────│ 监听Cookie变化  │◀────│ 用户授权 │
└─────────┘     └──────────────┘     └─────────────────┘     └──────────┘
```

#### 3.1.2 关键代码分析

**浏览器初始化**:
```python
def setup_browser(self):
    """配置浏览器选项"""
    co = ChromiumOptions()
    co.set_paths(browser_path=chrome_path, user_data_path=user_data_dir)
    co.set_argument(f'--profile-directory={active_profile}')
    
    # 平台特定选项
    if sys.platform.startswith('linux'):
        co.set_argument('--no-sandbox')
        co.set_argument('--disable-dev-shm-usage')
    elif sys.platform == 'darwin':
        co.set_argument('--disable-gpu-compositing')
    
    self.browser = ChromiumPage(co)
```

**Cookie 监听与 Token 提取**:
```python
def _wait_for_auth(self):
    """等待认证完成并提取 Token"""
    while time.time() - start_time < max_wait:
        cookies = self.browser.cookies()
        
        for cookie in cookies:
            if cookie.get("name") == "WorkosCursorSessionToken":
                value = cookie.get("value", "")
                # 提取 Token: userId::token 格式
                if "::" in value:
                    token = value.split("::")[-1]
                elif "%3A%3A" in value:
                    token = value.split("%3A%3A")[-1]
                return {"email": email, "token": token}
```

#### 3.1.3 智能重试机制

当检测到账号使用量达到上限时，自动删除账号并重新注册：

```python
def check_usage_limits(usage_str):
    """检查使用量是否超限"""
    parts = usage_str.split('/')
    current = int(parts[0].strip())
    limit = int(parts[1].strip())
    return (limit == 50 and current >= 50) or (limit == 150 and current >= 150)

# 如果超限，自动删除并重新注册
if check_usage_limits(usage_text):
    if self._delete_current_account():
        return self.handle_google_auth()  # 递归重试
```

### 3.2 机器码重置模块 (`reset_machine_manual.py` / `totally_reset_cursor.py`)

两套脚本均定义 **`MachineIDResetter`** 及 **`modify_main_js` / `modify_workbench_js`**，逻辑高度相似：菜单中「重置机器码」通常走 `reset_machine_manual.py`，「完全重置 Cursor」走 `totally_reset_cursor.py`（范围与步骤以各自实现为准）。

#### 3.2.1 重置流程

```
┌─────────────────────────────────────────────────────────────┐
│                     Machine ID 重置流程                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 备份原始文件                                             │
│     ├── storage.json.bak                                    │
│     └── state.vscdb.bak                                     │
│                                                             │
│  2. 生成新标识符                                             │
│     ├── devDeviceId: UUID v4                                │
│     ├── machineId: SHA256 (64字符)                          │
│     ├── macMachineId: SHA512 (128字符)                      │
│     └── sqmId: {UPPER_UUID}                                 │
│                                                             │
│  3. 更新数据文件                                             │
│     ├── storage.json (JSON)                                 │
│     └── state.vscdb (SQLite)                                │
│                                                             │
│  4. 更新系统级 ID                                            │
│     ├── Windows: MachineGuid, SQMClient/MachineId          │
│     └── macOS: platform.uuid.plist                          │
│                                                             │
│  5. 修改 Cursor 主程序                                       │
│     └── patch getMachineId() 函数                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 3.2.2 ID 生成算法

```python
def generate_new_ids(self):
    """生成新的机器标识符"""
    # UUID v4 - 设备ID
    dev_device_id = str(uuid.uuid4())
    
    # SHA256 - 机器ID (64字符十六进制)
    machine_id = hashlib.sha256(os.urandom(32)).hexdigest()
    
    # SHA512 - MAC机器ID (128字符十六进制)
    mac_machine_id = hashlib.sha512(os.urandom(64)).hexdigest()
    
    # 大括号格式的 UUID - SQM ID
    sqm_id = "{" + str(uuid.uuid4()).upper() + "}"
    
    return {
        "telemetry.devDeviceId": dev_device_id,
        "telemetry.macMachineId": mac_machine_id,
        "telemetry.machineId": machine_id,
        "telemetry.sqmId": sqm_id,
        "storage.serviceMachineId": dev_device_id,
    }
```

#### 3.2.3 SQLite 数据库操作

```python
def update_sqlite_db(self, new_ids):
    """更新 SQLite 数据库中的机器ID"""
    conn = sqlite3.connect(self.sqlite_path)
    cursor = conn.cursor()
    
    # 创建表结构
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS ItemTable (
            key TEXT PRIMARY KEY,
            value TEXT
        )
    """)
    
    # 使用 UPSERT 语法
    for key, value in new_ids.items():
        cursor.execute("""
            INSERT OR REPLACE INTO ItemTable (key, value) 
            VALUES (?, ?)
        """, (key, value))
    
    conn.commit()
    conn.close()
```

### 3.3 认证数据管理模块 (cursor_auth.py)

#### 3.3.1 数据存储结构

Cursor 使用 SQLite 数据库存储认证信息：

```sql
-- 表结构: ItemTable
CREATE TABLE ItemTable (
    key TEXT PRIMARY KEY,
    value TEXT
);

-- 关键键值对
| key                          | value              |
|------------------------------|-------------------|
| cursorAuth/cachedSignUpType  | Auth_0            |
| cursorAuth/cachedEmail       | user@example.com  |
| cursorAuth/accessToken       | eyJhbG...         |
| cursorAuth/refreshToken      | eyJhbG...         |
```

#### 3.3.2 事务处理

`update_auth` **未使用** `ON CONFLICT`。实现为：在 `BEGIN TRANSACTION` 后对每个 key 执行 `SELECT COUNT(*)`，不存在则 `INSERT`，存在则 `UPDATE`，最后 `COMMIT`；异常时 `ROLLBACK`。（机器码模块里的 `update_sqlite_db` 则使用 `INSERT OR REPLACE`，二者策略不同。）

```python
cursor.execute("BEGIN TRANSACTION")
try:
    for key, value in updates:
        cursor.execute("SELECT COUNT(*) FROM ItemTable WHERE key = ?", (key,))
        if cursor.fetchone()[0] == 0:
            cursor.execute(
                "INSERT INTO ItemTable (key, value) VALUES (?, ?)", (key, value)
            )
        else:
            cursor.execute(
                "UPDATE ItemTable SET value = ? WHERE key = ?", (value, key)
            )
    cursor.execute("COMMIT")
except Exception as e:
    cursor.execute("ROLLBACK")
    raise e
```

### 3.4 配置管理模块 (config.py)

#### 3.4.1 配置体系

```
┌─────────────────────────────────────────────────────────────┐
│                      配置层级结构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  默认配置 (代码内嵌)                                          │
│  ├── Chrome.chromepath                                      │
│  ├── Turnstile.* (验证相关等待)                              │
│  ├── Timing.* (各种等待时间)                                 │
│  └── Utils.enabled_update_check / enabled_account_info       │
│                                                             │
│  系统特定路径 (自动生成)                                       │
│  ├── WindowsPaths.*  ← Windows 注册表和文件路径              │
│  ├── MacPaths.*      ← macOS Application Support 路径        │
│  └── LinuxPaths.*    ← Linux ~/.config 下 Cursor/cursor      │
│                                                             │
│  用户配置 (Documents 下，见下)                                 │
│  └── 覆盖默认配置                                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

#### 3.4.2 跨平台路径处理

```python
def setup_config(translator=None):
    if sys.platform == "win32":
        default_config['WindowsPaths'] = {
            'storage_path': os.path.join(
                os.getenv("APPDATA"), 
                "Cursor", "User", "globalStorage", "storage.json"
            ),
            'sqlite_path': os.path.join(
                os.getenv("APPDATA"), 
                "Cursor", "User", "globalStorage", "state.vscdb"
            ),
            'machine_id_path': os.path.join(
                os.getenv("APPDATA"), "Cursor", "machineId"
            ),
        }
    elif sys.platform == "darwin":
        default_config['MacPaths'] = {
            'storage_path': os.path.expanduser(
                "~/Library/Application Support/Cursor/User/globalStorage/storage.json"
            ),
        }
    elif sys.platform == "linux":
        # 处理 sudo 情况
        sudo_user = os.environ.get('SUDO_USER')
        actual_home = f"/home/{sudo_user}" if sudo_user else os.path.expanduser("~")
```

**用户配置文件实际路径**：由 `utils.get_user_documents_path()` 与 `config.setup_config()` 决定，目录为 **`{用户文档目录}/.cursor-free-vip/config.ini`**。Windows 下即 `%USERPROFILE%\Documents\.cursor-free-vip\config.ini`；Linux/macOS 代码中同样使用 **`~/Documents/.cursor-free-vip/config.ini`**（不是家目录下的 `~/.cursor-free-vip`）。

### 3.5 其他业务模块与 main.py 菜单对应关系

`main.py` 中菜单项（编号随版本可能增减）与模块的大致对应：

| 功能 | 主要模块 |
|------|----------|
| 重置机器码 | `reset_machine_manual` |
| 自动注册（旧流程） | `cursor_register` / `new_signup` 等 |
| Google / GitHub OAuth 注册 | `cursor_register_google` / `cursor_register_github` → `oauth_auth` |
| 手动注册 | `cursor_register_manual` |
| 临时 GitHub 注册 | `github_cursor_register`（Selenium） |
| 退出 Cursor | `quit_cursor` |
| 禁用自动更新 | `disable_auto_update` |
| 完全重置 | `totally_reset_cursor` |
| 删除 Google 账号 | `delete_cursor_google` |
| 绕过版本检查 | `bypass_version` |
| 账号信息（可选） | `cursor_acc_info`（受 `Utils.enabled_account_info` 控制） |
| 启动时更新检查 | `main.check_latest_version`（`Utils.enabled_update_check`） |

---

## 4. 技术实现细节

### 4.1 浏览器自动化技术

**分工**：`oauth_auth.py` 中 **`OAuthHandler`** 使用 **DrissionPage**（`ChromiumPage` / `ChromiumOptions`）完成 Google、GitHub 在 `authenticator.cursor.sh` 上的流程；**`github_cursor_register.py`** 等路径使用 **Selenium** + `webdriver_manager`，与主 OAuth 栈分离。

#### 4.1.1 DrissionPage 优势

相比纯 Selenium，主路径使用 DrissionPage 提供了：
- **更简洁的 API**: `browser.ele("xpath://...")` 替代冗长的 WebDriverWait
- **更好的性能**: 直接操作 Chrome DevTools Protocol
- **原生 cookie 访问**: `browser.cookies()` 直接获取所有 cookie

#### 4.1.2 Chrome 配置策略

```python
def _configure_browser_options(self, chrome_path, user_data_dir, active_profile):
    co = ChromiumOptions()
    co.set_paths(browser_path=chrome_path, user_data_path=user_data_dir)
    co.set_argument(f'--profile-directory={active_profile}')
    
    # 基础配置
    co.set_argument('--no-first-run')
    co.set_argument('--no-default-browser-check')
    co.set_argument('--disable-gpu')
    
    # Linux 特定 (无头服务器环境)
    if sys.platform.startswith('linux'):
        co.set_argument('--no-sandbox')
        co.set_argument('--disable-dev-shm-usage')
        co.set_argument('--disable-setuid-sandbox')
    
    # Windows 特定
    elif os.name == 'nt':
        co.set_argument('--disable-features=TranslateUI')
        co.set_argument('--disable-features=RendererCodeIntegrity')
```

### 4.2 代码注入技术

#### 4.2.1 Cursor 主程序 Patch

为了绕过机器码检测，项目修改 Cursor 的 `main.js`：

```python
def modify_main_js(main_path, translator):
    """修改 main.js 文件中的 getMachineId 函数"""
    
    # 原始代码模式: 从系统读取机器ID
    # async getMachineId(){return a??b()}
    
    # 目标代码模式: 直接返回默认值，跳过系统读取
    # async getMachineId(){return b()}
    
    patterns = {
        r"async getMachineId\(\)\{return [^??]+\?\?([^}]+)\}": 
            r"async getMachineId(){return \1}",
        r"async getMacMachineId\(\)\{return [^??]+\?\?([^}]+)\}": 
            r"async getMacMachineId(){return \1}",
    }
    
    for pattern, replacement in patterns.items():
        content = re.sub(pattern, replacement, content)
```

#### 4.2.2 UI 修改

修改 `workbench.desktop.main.js` 来隐藏升级提示：

```python
def modify_workbench_js(file_path, translator):
    replacements = {
        # 修改 "Upgrade to Pro" 按钮为 GitHub 链接
        'title:"Upgrade to Pro",size:"small"': 
            'title:"wellssahara GitHub",size:"small",get onClick(){return function(){window.open("https://github.com/wellssahara/cursor-free-vip","_blank")}}}',
        
        # 修改 Pro Trial 为 Pro
        '<div>Pro Trial': '<div>Pro',
        
        # 隐藏通知 toast
        'notifications-toasts': 'notifications-toasts hidden',
    }
```

### 4.3 随机化技术

为了避免被检测为自动化工具，在 **`utils.py`** 中实现 **`get_random_wait_time(config, timing_key)`**：从 `config['Timing']` 读取字符串（支持 `0.5-1.5`、`0.5,1.5` 或单值），返回 `random.uniform`；解析失败时回退到默认区间。

```python
def get_random_wait_time(config, timing_key):
    """获取随机等待时间
    
    配置格式:
    - "0.5-1.5"  → 范围随机
    - "0.5,1.5"  → 范围随机 (逗号分隔)
    - "1.0"      → 固定值
    """
    timing = config.get('Timing', {}).get(timing_key)
    
    if '-' in timing:
        min_time, max_time = map(float, timing.split('-'))
    elif ',' in timing:
        min_time, max_time = map(float, timing.split(','))
    else:
        min_time = max_time = float(timing)
    
    return random.uniform(min_time, max_time)
```

---

## 5. 数据流分析

### 5.1 OAuth 注册数据流

```
┌─────────────┐
│  用户选择    │
│ Google/GitHub│
└──────┬──────┘
       │
       ▼
┌─────────────┐     ┌──────────────────┐
│  kill 现有   │────▶│  关闭 Chrome     │
│ Chrome 进程  │     │  进程            │
└──────┬──────┘     └──────────────────┘
       │
       ▼
┌─────────────┐     ┌──────────────────┐
│  启动 Chrome │────▶│  加载指定 Profile │
│  浏览器      │     │                  │
└──────┬──────┘     └──────────────────┘
       │
       ▼
┌─────────────┐     ┌──────────────────┐
│ 访问注册页面 │────▶│ authenticator.   │
│             │     │ cursor.sh/sign-up│
└──────┬──────┘     └──────────────────┘
       │
       ▼
┌─────────────┐     ┌──────────────────┐
│ 点击 OAuth  │────▶│  跳转 Google/    │
│ 按钮        │     │  GitHub 授权页   │
└──────┬──────┘     └──────────────────┘
       │
       ▼
┌─────────────┐     ┌──────────────────┐
│ 用户完成     │────▶│  用户手动登录    │
│ 授权        │     │  并授权          │
└──────┬──────┘     └──────────────────┘
       │
       ▼
┌─────────────┐     ┌──────────────────┐
│ 监听 Cookie │────▶│  轮询检查        │
│ 变化        │     │  WorkosCursor    │
│             │     │  SessionToken    │
└──────┬──────┘     └──────────────────┘
       │
       ▼
┌─────────────┐     ┌──────────────────┐
│ 提取 Token  │────▶│  解析 cookie     │
│             │     │  value 格式:     │
│             │     │  userId::token   │
└──────┬──────┘     └──────────────────┘
       │
       ▼
┌─────────────┐     ┌──────────────────┐
│ 保存到本地   │────▶│  写入 SQLite:    │
│ 数据库      │     │  state.vscdb     │
└──────┬──────┘     └──────────────────┘
       │
       ▼
┌─────────────┐
│   完成      │
└─────────────┘
```

### 5.2 机器码重置数据流

```
开始重置
    │
    ▼
┌─────────────────────────────────────────────┐
│ 1. 读取现有配置                              │
│    - storage.json                           │
│    - state.vscdb                            │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 2. 创建备份                                  │
│    - storage.json.bak                       │
│    - state.vscdb.bak                        │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 3. 生成新 ID                                 │
│    ├─ devDeviceId: uuid4()                  │
│    ├─ machineId: sha256(random)             │
│    ├─ macMachineId: sha512(random)          │
│    └─ sqmId: {UUID_UPPER}                   │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 4. 更新 storage.json                         │
│    - JSON 格式                              │
│    - 更新 telemetry.* 字段                  │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 5. 更新 state.vscdb                          │
│    - SQLite 事务                            │
│    - INSERT OR REPLACE                      │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 6. 更新系统注册表 (Windows)                   │
│    ├─ HKEY_LOCAL_MACHINE\SOFTWARE\          │
│    │  Microsoft\Cryptography\MachineGuid    │
│    └─ HKEY_LOCAL_MACHINE\SOFTWARE\          │
│       Microsoft\SQMClient\MachineId         │
└─────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────┐
│ 7. Patch Cursor 主程序                       │
│    - 修改 main.js                           │
│    - 修改 workbench.desktop.main.js         │
└─────────────────────────────────────────────┘
    │
    ▼
  完成
```

---

## 6. 跨平台实现

### 6.1 平台检测与适配

仓库中**没有**统一的 `get_platform()` 工具函数；实际代码在 **`config.py`**、**`main.py`**、**`reset_machine_manual.py`**、**`cursor_auth.py`** 等处直接使用 **`sys.platform`**（`win32` / `darwin` / `linux`）或 **`platform.system()`**。下面为与上述判断等价的参考写法：

```python
import platform
import sys

# 参考：与项目内分散判断一致
if sys.platform == "win32":
    # Windows
    pass
elif sys.platform == "darwin":
    # macOS
    pass
elif sys.platform.startswith("linux"):
    # Linux（含对 SUDO_USER、Cursor/cursor 目录名的兼容）
    pass
else:
    raise OSError(f"Unsupported platform: {platform.system()}")
```

### 6.2 各平台路径对照

| 数据类型 | Windows | macOS | Linux |
|----------|---------|-------|-------|
| **storage.json** | `%APPDATA%/Cursor/User/globalStorage/storage.json` | `~/Library/Application Support/Cursor/User/globalStorage/storage.json` | `~/.config/cursor/User/globalStorage/storage.json` |
| **state.vscdb** | `%APPDATA%/Cursor/User/globalStorage/state.vscdb` | `~/Library/Application Support/Cursor/User/globalStorage/state.vscdb` | `~/.config/cursor/User/globalStorage/state.vscdb` |
| **machineId** | `%APPDATA%/Cursor/machineId` | `~/Library/Application Support/Cursor/machineId` | `~/.config/cursor/machineid` |
| **Cursor 安装目录** | `%LOCALAPPDATA%/Programs/Cursor/resources/app` | `/Applications/Cursor.app/Contents/Resources/app` | 见下（多候选路径） |

Linux 下安装路径由 **`utils.get_linux_cursor_path()`** 在多个候选中选取第一个存在的目录，例如 `/opt/Cursor/resources/app`、`/usr/share/cursor/resources/app`、`~/.local/share/cursor/resources/app` 等；**`config.py`** 在初始化 `LinuxPaths` 时也会探测 `Cursor` / `cursor` 配置目录，与上表中的 `~/.config/cursor/...` 可能并存，以本机生成的 `config.ini` 为准。

### 6.3 Windows 注册表操作

```python
import winreg

def _update_windows_machine_guid(self):
    """更新 Windows MachineGuid"""
    key = winreg.OpenKey(
        winreg.HKEY_LOCAL_MACHINE,
        "SOFTWARE\\Microsoft\\Cryptography",
        0,
        winreg.KEY_WRITE | winreg.KEY_WOW64_64KEY
    )
    new_guid = str(uuid.uuid4())
    winreg.SetValueEx(key, "MachineGuid", 0, winreg.REG_SZ, new_guid)
    winreg.CloseKey(key)

def _update_windows_machine_id(self):
    """更新 Windows SQM MachineId"""
    new_guid = "{" + str(uuid.uuid4()).upper() + "}"
    
    key = winreg.OpenKey(
        winreg.HKEY_LOCAL_MACHINE,
        r"SOFTWARE\Microsoft\SQMClient",
        0,
        winreg.KEY_WRITE | winreg.KEY_WOW64_64KEY
    )
    winreg.SetValueEx(key, "MachineId", 0, winreg.REG_SZ, new_guid)
    winreg.CloseKey(key)
```

### 6.4 macOS 系统命令

```python
def _update_macos_platform_uuid(self, new_ids):
    """更新 macOS Platform UUID"""
    uuid_file = "/var/root/Library/Preferences/SystemConfiguration/com.apple.platform.uuid.plist"
    
    if os.path.exists(uuid_file):
        # 使用 sudo 执行 plutil 命令
        cmd = f'sudo plutil -replace "UUID" -string "{new_ids["telemetry.macMachineId"]}" "{uuid_file}"'
        result = os.system(cmd)
        if result == 0:
            print("macOS Platform UUID updated successfully")
```

---

## 7. 安全与防护机制

### 7.1 反检测策略

| 策略 | 实现方式 | 目的 |
|------|----------|------|
| **随机延迟** | `random.uniform(min, max)` | 模拟人类操作间隔 |
| **真实浏览器** | 使用用户现有 Chrome Profile | 保留真实用户数据 |
| **Cookie 复用** | 从已登录浏览器提取 | 跳过重复登录验证 |
| **进程清理** | `taskkill` / `pkill` | 关闭冲突进程 |

### 7.2 备份机制

每次修改关键文件前自动创建备份：

```python
def reset_machine_ids(self):
    # 创建备份
    backup_path = self.db_path + ".bak"
    if not os.path.exists(backup_path):
        shutil.copy2(self.db_path, backup_path)
    
    # 执行修改...
```

### 7.3 权限检查

```python
# 检查管理员权限 (Windows)
def is_admin():
    try:
        return ctypes.windll.shell32.IsUserAnAdmin() != 0
    except:
        return False

# 检查文件读写权限
if not os.access(self.db_path, os.R_OK | os.W_OK):
    print("Permission denied. Please run as administrator.")
    return False
```

---

## 8. 构建与部署

### 8.1 构建配置 (build.spec)

`build.spec` 通过 **`load_dotenv()`** 读取环境变量 **`VERSION`**（默认 `1.0.0`），并依 **`platform.system()`** 生成可执行文件名前缀 **`CursorFreeVIP_{VERSION}_{windows|linux|mac}`**。以下为与仓库一致的 **`Analysis`** 摘要（完整内容以根目录 `build.spec` 为准）：

```python
import os
from dotenv import load_dotenv

load_dotenv()
version = os.getenv('VERSION', '1.0.0')
output_name = f"CursorFreeVIP_{version}_{os_type}"  # os_type: windows / linux / mac

a = Analysis(
    ['main.py'],
    pathex=[],
    binaries=[],
    datas=[
        ('turnstilePatch', 'turnstilePatch'),
        ('PBlock', 'PBlock'),
        ('locales', 'locales'),
        ('cursor_auth.py', '.'),
        ('reset_machine_manual.py', '.'),
        ('cursor_register.py', '.'),
        ('new_signup.py', '.'),
        ('new_tempemail.py', '.'),
        ('quit_cursor.py', '.'),
        ('cursor_register_manual.py', '.'),
        ('oauth_auth.py', '.'),
        ('utils.py', '.'),
        ('.env', '.'),
        ('block_domain.txt', '.'),
    ],
    hiddenimports=[
        'cursor_auth',
        'reset_machine_manual',
        'new_signup',
        'new_tempemail',
        'quit_cursor',
        'cursor_register_manual',
        'oauth_auth',
        'utils',
    ],
    # ... hookspath、excludes 等与仓库一致；本仓库未使用 cipher=block_cipher
)
# EXE(name=output_name, ...) 控制台程序；可选环境变量 TARGET_ARCH 指定架构
```

**说明**：`colorama`、`requests`、`psutil`、`DrissionPage` 等依赖通常由 PyInstaller 从 **`main.py` 及动态导入** 的分析链自动收集；`hiddenimports` 主要补齐通过字符串/动态加载的模块。若打包后缺模块，再按需追加 `hiddenimports`。

### 8.2 构建流程

```
┌─────────────────────────────────────────────────────────────┐
│                       构建流程                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 清理缓存                                                 │
│     └── rm -rf build/ dist/                                 │
│                                                             │
│  2. 读取版本号                                               │
│     └── 从 .env 或代码中读取 VERSION                        │
│                                                             │
│  3. 执行 PyInstaller                                        │
│     └── pyinstaller --clean --noconfirm build.spec          │
│                                                             │
│  4. 打包输出                                                 │
│     └── CursorFreeVIP_{VERSION}_{windows|linux|mac}        │
│         （无扩展名或依平台为 .exe；VERSION 来自 .env）        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 分发脚本

项目提供了一键安装脚本：

**Windows (PowerShell)**:
```powershell
irm https://raw.githubusercontent.com/wellssahara/cursor-free-vip/main/scripts/install.ps1 | iex
```

**Linux/macOS (Bash)**:
```bash
curl -fsSL https://raw.githubusercontent.com/wellssahara/cursor-free-vip/main/scripts/install.sh -o install.sh && chmod +x install.sh && ./install.sh
```

---

## 9. 关键技术要点总结

### 9.1 核心技术亮点

1. **多平台统一抽象**: 通过 `config.py` 中的平台检测，为不同操作系统提供统一的接口

2. **浏览器自动化最佳实践**: 
   - 使用 DrissionPage 替代纯 Selenium，提高稳定性
   - 复用用户现有 Chrome Profile，减少被检测风险
   - 智能等待机制，避免硬编码 sleep

3. **数据持久化策略**:
   - SQLite 用于结构化认证数据
   - JSON 用于配置存储
   - INI 用于用户自定义配置

4. **安全防护**:
   - 每次修改前自动备份
   - 事务机制确保数据库一致性
   - 权限检查避免操作失败

### 9.2 可改进方向

1. **架构层面**:
   - 引入依赖注入框架，提高可测试性
   - 使用 asyncio 替代多线程，提升并发性能

2. **安全层面**:
   - 添加加密存储敏感信息
   - 实现操作日志记录

3. **用户体验**:
   - 添加 GUI 界面
   - 增强内置更新流程（例如镜像、校验与回滚；当前已有 GitHub Release 检查与安装脚本拉起）

---

## 附录

### A. 依赖清单

```
watchdog              # 文件监控
python-dotenv>=1.0.0  # 环境变量
colorama>=0.4.6       # 终端颜色
requests              # HTTP 请求
psutil>=5.8.0         # 进程管理
pywin32               # Windows API (Windows only)
pyinstaller           # 打包工具
DrissionPage>=4.0.0   # 浏览器自动化
selenium              # WebDriver 支持
webdriver_manager     # 驱动管理
```

### B. 文件位置速查

| 说明 | 路径或文件 |
|------|------------|
| 用户配置目录 | `{用户文档}/.cursor-free-vip/`（Windows：`%USERPROFILE%\Documents\.cursor-free-vip`；类 Unix：`~/Documents/.cursor-free-vip`） |
| 用户配置文件 | 同上目录下的 `config.ini` |
| 主程序入口 | `main.py` |
| OAuth（DrissionPage） | `oauth_auth.py` |
| Selenium GitHub 注册 | `github_cursor_register.py` |
| 认证写入 DB | `cursor_auth.py` |
| 机器码重置（常用） | `reset_machine_manual.py` |
| 完全重置 | `totally_reset_cursor.py` |
| 构建规格 | `build.spec` |
| 域名列表（打包） | `block_domain.txt` |
| 语言包 | `locales/*.json` |
| Turnstile 扩展目录 | `turnstilePatch/` |
| 请求拦截扩展目录 | `PBlock/` |

---

*文档版本: 1.1*  
*更新日期: 2026-03-26*  
*项目地址: https://github.com/wellssahara/cursor-free-vip*
