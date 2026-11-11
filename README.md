# Taygedo Auto Sign

> 塔吉多（老虎平台）社区自动签到 Android 应用 — 多账号 · 定时签到 · 零依赖构建

<p align="center">
  <img src="favicon.ico" width="96" alt="logo"/>
</p>

---

## 功能特性

- ✅ **多账号管理** — 支持同时保存多个塔吉多账号，一键逐个签到或全量签到
- ⏰ **定时自动签到** — 基于 `AlarmManager` 定时触发自启动，无需手动干预，建议搭配手机自动任务
- 🔄 **Token 自动刷新** — accessToken 过期后自动用 refreshToken 换新，无需重新登录
- 📧 **邮件通知** — 签到结果可通过自定义 SMTP 邮箱发送（支持 SSL / STARTTLS / AUTH LOGIN）
- 🔔 **系统通知** — 每次定时签到完成后推送 Android 通知，汇总成功 / 失败账号
- 📦 **应用自更新** — Gitee `version.json` 静态文件版本检测，APK 内下载并静默安装
- 🎯 **零依赖构建** — 不依赖 Gradle / Android Studio，纯命令行工具链 + Python 脚本一键出包

---

## 项目结构

```
Taygedo Auto Sign（1.2.2 apk源码）/
├── android-build/              # APK 构建系统（完整独立）
│   ├── app/                    # ★ Java 源码 & AndroidManifest.xml
│   │   ├── MainActivity.java       主界面（登录 / 签到 / 账号管理）
│   │   ├── LoginActivity.java      登录页
│   │   ├── MoreActivity.java       更多设置页
│   │   ├── AddSenderActivity.java  添加发件箱
│   │   ├── AndroidManifest.xml     清单文件
│   │   ├── AccountStore.java       账号持久化（SharedPreferences + JSON）
│   │   ├── LoginClient.java        塔吉多 / 老虎平台完整 HTTP 客户端
│   │   ├── EmailSender.java        纯手写 SMTP 邮件客户端
│   │   ├── SignReceiver.java       AlarmManager 定时签到接收器
│   │   ├── BootReceiver.java       开机自启动接收器
│   │   ├── UpdateChecker.java      Gitee 静态 JSON 版本检测器
│   │   ├── DailySignTracker.java   每日签到记录追踪
│   │   ├── CryptoUtils.java        设备 ID 生成 & 加密工具
│   │   └── LocalFileProvider.java  极简 FileProvider（APK 安装授权）
│   ├── jdk/jdk-17.0.20.1+1/    # 内嵌 JDK 17
│   ├── sdk/                    # 内嵌 Android SDK
│   │   ├── platforms/android-33/
│   │   └── build-tools/34.0.0/
│   ├── build.py                # ★ 构建入口脚本（Python 3）
│   ├── debug.keystore          # 调试签名密钥（自动生成）
│   └── work/                   # 构建中间产物（自动生成）
├── ui-builder/
│   └── index.html              # UI 可视化设计器（三栏布局预览器）
├── favicon.ico                 # 应用图标源（build.py 自动转多种 mipmap 尺寸）
├── version.json                # 版本信息（自更新检测拉取的静态文件）
└── README.md                   # 本文件
```

---

## 构建指南

### 环境要求

| 项目 | 要求 |
|------|------|
| 操作系统 | **Windows**（构建脚本使用了 Windows 专属的 8.3 短路径和 NTFS junction） |
| Python | 3.8+（仅使用标准库 + 可选的 Pillow 用于图标生成） |
| 其他 | **无需安装 Android Studio / Gradle / Java**，构建工具链全部内嵌 |

### 一键构建

```powershell
cd "android-build"
python build.py
```

构建完成后 APK 位于 `android-build/output/TaygedoAutoSign.apk`。

### 构建流程（build.py 内部步骤）

```
Step 1  准备工作目录 / 清理旧产物
Step 2  aapt2 compile + link  →  基础 APK 壳
Step 3  javac 编译 Java 源码  →  .class
Step 4  d8  .class → .dex
Step 5  把 classes.dex 塞进 APK（zip 追加）
Step 6  zipalign 4 字节对齐
Step 7  首次自动生成 debug.keystore
Step 8  apksigner 签名 + 验证
```

### 为什么需要 `C:\nte_build` junction？

Android 原生工具（`aapt2` / `d8` / `javac`）对含中文或特殊字符的 Windows 路径支持极差。`build.py` 启动时会自动创建一个 NTFS junction 链接 `C:\nte_build` 指向实际构建目录，所有工具调用都经由这个纯 ASCII 路径，完美绕过编码问题。

---

## 技术亮点

### 1. 不依赖 `javax.mail` 的纯手写 SMTP 客户端（[EmailSender.java](android-build/app/EmailSender.java)）

去掉了 `javax.mail` + `javax.activation` 依赖，APK 体积减少约 **700 KB**，同时避免了 Android 上 `java.awt` / `ServiceLoader` 的兼容性问题。

### 2. 完整复刻的 HTTP 协议层（[LoginClient.java](android-build/app/LoginClient.java)）

从 `NTEMaid` JAR 反编译提取，包含老虎平台 → 塔吉多换票、角色绑定、签到、token 刷新等完整流程。所有设备签名参数硬编码与原应用保持一致，确保接口不会因签名算法差异返回 401。

### 3. 完全程序化构建的 UI

应用 **不包含任何 XML 布局文件**，所有界面元素（Activity 的 `setContentView` 内部全部使用 `LinearLayout` / `TextView` / `Button` 动态创建）。`android-build/build.py` 仅用约 30 行代码自动生成 `res/values/strings.xml` 和 `res/xml/file_paths.xml`。

### 4. 零 API 调用的自更新（[UpdateChecker.java](android-build/app/UpdateChecker.java)）

自更新检测走 `version.json` 静态 JSON 文件，**零 GitHub API 调用、零 Token、零速率限制**。

### 5. 独立可复现的构建链

整个 `android-build/` 目录是完全自包含的：JDK 17 + Android SDK Platform 33 + Build-Tools 34.0.0 + Python 构建脚本。拿到这个目录，在 Windows 上直接 `python build.py` 就能出 APK，不需要任何额外环境配置。

---

## 依赖说明

| 依赖 | 用途 | 位置 |
|------|------|------|
| JDK 17 (内嵌) | javac / keytool / jarsigner | `android-build/jdk/jdk-17.0.20.1+1/` |
| Android SDK (内嵌) | aapt2 / d8 / zipalign / apksigner / android.jar | `android-build/sdk/` |
| JSON (org.json) | 随 Android SDK 提供，编译时通过 android.jar 引入 | — |
| Pillow (可选) | build.py 自动生成 mipmap 图标 | PyPI（构建时可选） |

---

## 系统权限说明

| 权限 | 用途 |
|------|------|
| `INTERNET` | 访问塔吉多 / 老虎平台 API |
| `ACCESS_NETWORK_STATE` | 检查网络可用性 |
| `RECEIVE_BOOT_COMPLETED` | 开机后恢复定时签到任务 |
| `WAKE_LOCK` | 签到期间保持设备唤醒 |
| `SCHEDULE_EXACT_ALARM` | 精确时间点触发签到（Android 12+） |
| `VIBRATE` | 签到完成提示震动 |
| `REQUEST_INSTALL_PACKAGES` | 自更新时调起系统安装器 |

---

## 版本历史

参见 [version.json](version.json)。

---

## Taygedo Auto Sign v1.2.5

- 新增「签到成功后自动退出」开关（位于「更多设置」页），适合搭配第三方自动化工具使用，签到完成后自动关闭软件
- 开启后可设置生效时间段，精确到分钟，支持跨午夜的时间段
- 时间段内打开软件签到成功后，提示「已签到完成，将在10秒后自动关闭」，10 秒后自动关闭软件；有账号签到失败时不会退出
- 开启自动发送邮件功能时，会等邮件发送完毕后再提示并关闭
- 当天已签到或没有未签到的账号时，在时间段内打开软件同样按已签到完成处理
- 手动关闭该开关会立即打断自动关闭，软件不会退出；重新开启后需在下次启动软件时才会生效

## Taygedo Auto Sign v1.2.4

- 【重要修复】解决锁屏后第三方自动化工具启动 App 无法签到的问题，彻底修复 Doze 模式下 ETIMEDOUT / DNS 解析失败 / 连接超时等网络故障
- 【重要修复】解决首次签到失败后后续全部报「End of input」的连锁故障，禁用坏连接复用，防止网络状态损坏后一直失败直到重启
- 新增指数退避重试机制，网络异常自动重试 3 次（2s / 5s / 8s），弱网和 Doze 恢复场景签到成功率大幅提升
- 新增 DNS 预解析缓存，降低首次请求超时概率

## Taygedo Auto Sign v1.2.3

- 修复「更多设置」页面每次进入都弹出已开启提示的问题
- 修复锁屏 / Doze 模式下定时签到失败的问题，添加 WakeLock 保护
- 网络请求超时从 15 秒增加到 30 秒

## Taygedo Auto Sign v1.2.2

- 新增版本自动检查功能，可在应用内直接下载并安装更新
- 优化签到流程稳定性

安装包保存地址：Android/data/应用包名/files/Download
一般情况下安装新版本后会自动删除安装包，如未删除可手动删除。

## Taygedo Auto Sign v1.2.1

### 因 v1.2 版本存在自动登录失败 bug，现已下架修复此 bug 并上架 v1.2.1 版本

- 修复 v1.2 版本有概率自动登录失败的 bug
- 依旧轻量化软件，存储大小不到 1MB！
- 如需回退至 v1.1 版本，请卸载新版本后再下载旧版本使用

## Taygedo Auto Sign v1.2

- 新增自动发送邮件功能，可在接收邮箱查看详细签到情况
- 此版本后将不再控制软件功耗，相比 v1.1 版本功耗有所增加（但不多，依旧属于低功耗软件），可自主选择是否更新
- 如需回退至 v1.1 版本，请卸载新版本后再下载旧版本使用

## Taygedo Auto Sign v1.1

### 我们拒绝假自动，只要真自动！Taygedo Auto Sign v1.1 来啦~

- 新增自动登录功能，登陆一次，自动登录！（该功能仅支持密码登录）
- 依旧轻量化软件，存储大小不到 1MB！
- 修复部分历史遗留 bug，增强稳定性
- 软件名已更改为 Taygedo Auto Sign
- 此版本为功耗最优版本，适合低功耗需求用户，性能要求最低！

## NTE Auto Sign v1.0

- 页面干净简洁，存储占用小，不到 1MB！
- 功耗低，内存占用小！
- 签到更直观！哪天漏签都能看到！
- 支持多账号同时签到领奖励！
- 支持全自动签到！不用担心漏签问题！
- 支持密码 / 验证码登录！
- 支持定时启动（不稳定，如有需求建议用手机自带的自动任务启动软件）
- 该项目已于 V1.1 版本改名 Taygedo Auto Sign

---

## 免责声明

本项目暂不提供源码，仅供学习研究使用。塔吉多、老虎平台及相关游戏名称为其各自持有者的商标。请遵守目标平台的用户协议，合理使用自动化功能。
