# WorkBuddy 一键签到工具 · WorkBuddy Daily Check-in

> **简体中文** · **English** — 本文档提供中英双语版本，请按语言跳转。
>
> [🇨🇳 简体中文](#简体中文)  |  [🇬🇧 English](#english)

![Platform](https://img.shields.io/badge/platform-Windows%2010%2F11%20x64-blue)
![Flutter](https://img.shields.io/badge/Flutter-3.x-02569B)
![Dart](https://img.shields.io/badge/Dart-3.x-0175C2)
![Distribution](https://img.shields.io/badge/distribution-single--file--8.3MB-lightgrey)
![Dependencies](https://img.shields.io/badge/runtime-no--install--needed-brightgreen)

---

<a id="简体中文"></a>

## 简体中文

面向 Windows 的桌面签到工具，以**单个自包含 exe** 分发。内置「温和模式」：通过拟人化请求节律、请求头一致性、指数退避与风控熔断，让自有账号的正常每日签到更接近真人操作，而不是脚本行为。

> **边界**：项目不提供、也不尝试任何绕过验证码、风控校验或身份伪造的能力。命中限流或风控提示时立即停止请求并进入冷却，交由人工处理。

### 功能特性

| 能力 | 说明 |
| --- | --- |
| 单文件分发 | 一个 exe 内含程序、Flutter 运行时与 VC++ 运行库，无需安装 |
| 多账号管理 | 每账号独立凭据文件，批量串行签到 |
| 网页会话授权 | 内置 Edge 登录窗口；也支持授权码 / cURL / HAR / 裸 Cookie · Bearer |
| 温和模式 | 随机节流、首页预热、浏览器一致请求头、连接复用、退避、熔断、每日幂等锁 |
| 每日自动签到 | Windows 计划任务，最长 40 分钟随机延迟，静默运行 |
| 只读状态查询 | 带缓存的凭据 / 状态检查，不产生写操作 |
| 积分与旅行 | 余额、连续天数、本期累计，可选派发旅行任务 |

### 运行要求

| 项目 | 要求 |
| --- | --- |
| 操作系统 | Windows 10 / 11，x64 |
| 运行库 | **无需安装** — `flutter_windows.dll`、`data/` 与 VC++ CRT DLL 均已随包 |
| 首次授权 | Microsoft Edge（系统自带）+ 网络 |
| 日常运行 | 网络可达目标站点 + 有效凭据 |
| 不需要 | Flutter SDK、Dart、Python、.NET、Visual C++ Redistributable |

随包附带的 CRT 文件为 `MSVCP140.dll`、`VCRUNTIME140.dll`、`VCRUNTIME140_1.dll`（Microsoft Visual C++ Redistributable，按应用本地部署条款随附）。

### 快速开始

1. 从 Releases 下载 `WB一键签到.exe`（约 8.3 MB），放到任意目录。
2. 双击运行。首次启动会把运行时释放到 `%LOCALAPPDATA%\WBCheckin\bin\`（1~3 秒），之后复用。
3. 在面板选择「添加账号 / 网页授权」，在弹出的 Edge 窗口中用**本人账号**登录，再点「采集凭据」。
4. 执行一次只读状态刷新，确认凭据有效。

仓库与 exe 中**均不含**任何账号凭据。

### 单文件构成

```text
WB一键签到.exe
├── 7zSD.sfx    静默自解压引导（7-Zip / LZMA SDK 官方模块）
├── setup.bat   释放并启动的脚本
├── 7za.exe     7-Zip 独立解压器
└── payload.7z  wb_checkin.exe + flutter_windows.dll + data/ + VC++ CRT DLL
```

| 落盘位置 | 内容 |
| --- | --- |
| `%LOCALAPPDATA%\WBCheckin\bin\` | 仅程序本体（每次运行覆盖更新） |
| `%USERPROFILE%\Documents\WB一键签到\save\` | **全部**用户数据：凭据、状态、缓存、日志、浏览器配置 |

可用环境变量 `WB_CHECKIN_DATA_DIR` 覆盖数据目录。计划任务注册的是释放后的 `wb_checkin.exe`，因此临时解压目录被清理后任务依然有效。

### 数据目录

```text
%USERPROFILE%\Documents\WB一键签到\save\
├── creds\<账号>.json       会话凭据
├── state\cache.json        状态缓存（TTL 5 分钟）
├── state\cooldown.json     风控冷却（触发时生成）
├── state\last_run.json     每日幂等锁
├── last_run.log            无窗口运行日志
└── browser_profile\        内置登录浏览器配置
```

### 命令行与计划任务

```bat
wb_checkin.exe --checkin-all --auto               :: 全部账号签到，跑完退出
wb_checkin.exe --checkin-all --auto --check-only  :: 只读状态检查
```

在面板勾选「每天自动签到」即可，任务名 `WorkBuddyCheckinFlutter`，带最长 40 分钟随机延迟，结果写入 `last_run.log`。给单文件设置 `WB_CHECKIN_HEADLESS=1` 可跳过图形界面、静默执行只读检查。

### 温和模式

| 脚本特征 | 温和处理 |
| --- | --- |
| 请求间隔为 0，背靠背 | 相邻请求随机 1.2~3.6 秒；账号之间 6~18 秒 |
| 请求头稀少、顺序随机 | 按真实 UA 家族生成；Chromium 系带 `sec-ch-ua` 与 `Sec-Fetch-*`，Firefox 系不带 |
| 首个请求直接打业务接口 | 先 GET 首页完成预热 |
| 每次新建连接 | keep-alive 复用，代理场景自动回退 |
| 失败即反复重跑 | 仅 429 / 5xx / 网络错误做指数退避 + 抖动，遵守 `Retry-After` |
| 多账号并发 | 串行执行 + 随机间隔 |
| 每天同一秒触发 | 计划任务随机延迟 |
| 一天内重复点击 | 无人值守启用每日幂等锁；状态查询 5 分钟缓存 |

**熔断优先**：命中限流或风控文案立即停手并写入冷却，后续账号全部跳过。请等待冷却自然结束，不要在冷却期内重试。每次签到比直连慢约 10~20 秒，这是刻意设计。

### 安全与隐私

- 所有数据只留在本机，不上报、不埋点；请求仅发往用户自行配置的目标站点。
- 内置登录只读取完成签到所需的会话字段，不读取或存储密码。
- 遇验证码、二次验证或风控拦截一律停止并转人工。
- 仅用于本人合法持有的账号，并请遵守目标站点服务条款。

### 从源码构建

需要 Flutter 3.x、Visual Studio 2022（C++ 桌面工作负载）、7-Zip。

```bat
cd flutter-ver
flutter pub get
rebuild.bat
```

`rebuild.bat` 已处理两个 Windows 侧约束：

1. **非 ASCII 路径**会让 MSBuild 自定义步骤读不到 `app.dill`，脚本先把源码镜像到纯 ASCII 路径再构建。
2. `windows/runner/` 下**含中文的 C++ 源文件必须带 UTF-8 BOM**，否则 MSVC 报 `C4819` / `C2001`。

重新打包单文件：把 `Release\` 与三个 VC++ CRT DLL 压成 `payload.7z`，再与 `setup.bat`、`7za.exe` 合成 `outer.7z`，写 `config.txt`（`GUIMode="2"` + `RunProgram="setup.bat"`），最后按 `7zSD.sfx + config.txt + outer.7z` 顺序二进制拼接。完整命令见 `flutter-ver/构建说明.txt`。

### 故障排查

| 现象 | 处理 |
| --- | --- |
| HTTP 401 / 403 凭据失效 | 在面板重新授权；确认 UA 与采集时一致 |
| 提示「冷却中」 | 等待冷却结束并人工确认，不要反复重试 |
| 双击无反应 | 确认 exe 完整（约 8.3 MB）、未被安全软件拦截；首次运行需 1~3 秒 |
| 构建报中文路径乱码 | 使用 `rebuild.bat`，勿直接在中文路径下执行 `flutter build windows` |
| 构建报 `C4819` / `C2001` | 检查含中文的 runner 源文件是否带 UTF-8 BOM |

### 许可证与免责声明

**许可证**：当前未附带开源许可证，默认保留所有权利。若打算开放分发，请先添加 `LICENSE`（MIT / Apache-2.0 等）并同步修改本节。随包附带的 VC++ 运行库 DLL 与 7-Zip 组件按各自官方可再分发条款随应用本地部署。

**免责声明**：本项目按「现状」提供，仅用于个人学习与自动化技术实践。使用者须遵守目标站点服务条款与适用法规，并仅操作本人合法持有的账号。因使用本项目导致的账号限制、积分变动、数据丢失或任何间接损失，作者不承担责任。项目不包含绕过安全校验的能力。

---

<a id="english"></a>

## English

A Windows desktop tool for daily check-in, distributed as a **single self-contained executable**. It ships with a *Gentle Mode* that shapes request cadence, header consistency, exponential backoff and risk-control circuit breaking, so a personal account behaves closer to a human operator rather than a script.

> **Scope**: this project does **not** bypass CAPTCHA, risk checks or identity verification. On rate-limit or risk signals it stops immediately and enters a cooldown for human review.

### Highlights

| Feature | Description |
| --- | --- |
| Single-file distribution | One `.exe` bundles the app, Flutter runtime and VC++ runtime — nothing to install |
| Multi-account | One credential file per account, serial batch check-in |
| Web-session authorization | Built-in Edge login window; also accepts authorization codes / cURL / HAR / raw Cookie · Bearer |
| Gentle Mode | Random pacing, page warm-up, browser-consistent headers, keep-alive, backoff, circuit breaker, daily idempotency lock |
| Scheduled check-in | Windows Task Scheduler with up to 40 minutes of random delay, runs silently |
| Read-only status | Cached credential / status check without any write operation |
| Points & travel | Balance, streak, cycle totals, optional travel task |

### Requirements

| Item | Requirement |
| --- | --- |
| OS | Windows 10 / 11, x64 |
| Runtime | **None** — `flutter_windows.dll`, `data/` and the VC++ CRT DLLs are bundled |
| First authorization | Microsoft Edge (built into Windows) + network access |
| Daily use | Network access to the target site + valid credentials |
| Not required | Flutter SDK, Dart, Python, .NET, Visual C++ Redistributable |

The bundled CRT files are `MSVCP140.dll`, `VCRUNTIME140.dll` and `VCRUNTIME140_1.dll` (Microsoft Visual C++ Redistributable, app-local deployment).

### Quick start

1. Download `WB一键签到.exe` (~8.3 MB) from Releases and place it anywhere.
2. Double-click it. The first launch extracts the runtime to `%LOCALAPPDATA%\WBCheckin\bin\` (1–3 seconds); later launches reuse it.
3. In the panel choose **Add account / Web authorization**, sign in with **your own account** in the Edge window that opens, then click **Collect credentials**.
4. Run a read-only status refresh to confirm the credential is valid.

Credentials are **never** included in the repository or the executable.

### What is inside the single file

```text
WB一键签到.exe
├── 7zSD.sfx    Silent self-extracting stub (7-Zip / LZMA SDK)
├── setup.bat   Extract-and-launch script
├── 7za.exe     Standalone 7-Zip extractor
└── payload.7z  wb_checkin.exe + flutter_windows.dll + data/ + VC++ CRT DLLs
```

| On-disk location | Contents |
| --- | --- |
| `%LOCALAPPDATA%\WBCheckin\bin\` | Program binaries only (overwritten on each run) |
| `%USERPROFILE%\Documents\WB一键签到\save\` | **All** user data: credentials, state, cache, logs, browser profile |

Set `WB_CHECKIN_DATA_DIR` to override the data directory. Scheduled tasks are registered against the extracted `wb_checkin.exe`, so they remain valid after the temporary extraction folder is cleaned up.

### Data layout

```text
%USERPROFILE%\Documents\WB一键签到\save\
├── creds\<account>.json        Session credentials
├── state\cache.json            Status cache (5 min TTL)
├── state\cooldown.json         Risk cooldown (created on trigger)
├── state\last_run.json         Daily idempotency lock
├── last_run.log                Headless run log
└── browser_profile\            Built-in login browser profile
```

### Command line & scheduled task

```bat
wb_checkin.exe --checkin-all --auto               :: check in all accounts, then exit
wb_checkin.exe --checkin-all --auto --check-only  :: read-only status check
```

Enable **Daily auto check-in** in the panel. The task is named `WorkBuddyCheckinFlutter`, carries a random delay of up to 40 minutes, and writes results to `last_run.log`. Set `WB_CHECKIN_HEADLESS=1` when launching the single-file build to run a silent read-only check instead of opening the GUI.

### Gentle Mode

| Naive script pattern | Gentle handling |
| --- | --- |
| Zero-delay back-to-back requests | Random 1.2–3.6 s between requests; 6–18 s between accounts |
| Sparse / randomly ordered headers | Headers generated from the real UA family; Chromium gets `sec-ch-ua` + `Sec-Fetch-*`, Firefox does not |
| First request hits the business API | A GET to the page front door warms up the session first |
| New connection per request | keep-alive reuse, graceful fallback behind a proxy |
| Retry immediately on failure | Exponential backoff + jitter for 429 / 5xx / network errors only, honouring `Retry-After` |
| Accounts in parallel | Serial execution with random gaps |
| Same second every day | Scheduler random delay |
| Repeated clicks in one day | Daily idempotency lock in unattended mode; 5-minute status cache |

**Circuit breaker first**: on rate-limit or risk wording, all requests stop and a cooldown is written; remaining accounts are skipped. Wait for it to expire — do not retry inside the cooldown. Expect each check-in to take 10–20 seconds longer than a direct call; this is intentional.

### Security & privacy

- Everything stays on the local machine; no telemetry, no third-party upload. Requests go only to the site configured by the user.
- The built-in login reads only the session fields needed for check-in, never the password.
- CAPTCHA / 2FA / risk challenges always stop execution and defer to a human.
- Use only accounts you legitimately own, and check the target site's terms of service.

### Build from source

Requirements: Flutter 3.x, Visual Studio 2022 (C++ desktop workload), 7-Zip.

```bat
cd flutter-ver
flutter pub get
rebuild.bat
```

Two Windows-specific constraints are handled by `rebuild.bat`:

1. **Non-ASCII paths** break MSBuild custom steps (`app.dill` read failure), so the script mirrors sources to an ASCII-only path before building.
2. **C++ sources containing Chinese** under `windows/runner/` must carry a UTF-8 BOM, otherwise MSVC reports `C4819` / `C2001`.

To repackage the single file, archive `Release\` plus the three VC++ CRT DLLs into `payload.7z`, combine it with `setup.bat` and `7za.exe` into `outer.7z`, write a `config.txt` (`GUIMode="2"`, `RunProgram="setup.bat"`), then concatenate `7zSD.sfx + config.txt + outer.7z`. Full commands are in `flutter-ver/构建说明.txt`.

### Troubleshooting

| Symptom | Action |
| --- | --- |
| HTTP 401 / 403, credential expired | Re-authorize in the panel; verify the UA matches the one captured |
| "Cooldown" message | Wait for expiry and confirm manually; do not retry during cooldown |
| Nothing happens on double-click | Verify the exe is intact (~8.3 MB) and not blocked by security software; first launch needs 1–3 s |
| Build fails with garbled Chinese path | Use `rebuild.bat`, do not run `flutter build windows` directly in a non-ASCII path |
| Build fails with `C4819` / `C2001` | Ensure Chinese-containing runner sources have a UTF-8 BOM |

### License & disclaimer

**License**: no open-source license is bundled yet; all rights reserved by default. Add a `LICENSE` file (MIT / Apache-2.0, etc.) before release if redistribution is intended. The bundled VC++ runtime DLLs and 7-Zip components are redistributed under their respective official terms.

**Disclaimer**: provided "as is" for personal learning and automation practice. Users must comply with the target site's terms and applicable law, and use only accounts they own. The author is not liable for account restrictions, point changes, data loss or any indirect damages. This project contains no capability to bypass security checks.

---

<div align="center">
<sub>Windows · Flutter · Single-file · 简体中文 / English</sub>
</div>
