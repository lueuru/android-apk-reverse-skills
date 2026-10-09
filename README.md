<div align="center">

# Android APK 逆向与改造技能包

**不改一行字节码，也能改 APK：字符串就地替换、渠道 SDK 剥离、自动化验证**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](./LICENSE)
![Platform](https://img.shields.io/badge/platform-Android-3ddc84?style=flat-square&logo=android&logoColor=white)
![Lang](https://img.shields.io/badge/语言-简体中文-1f6feb?style=flat-square)
[![Last commit](https://img.shields.io/github/last-commit/lueuru/android-apk-reverse-skills?style=flat-square)](https://github.com/lueuru/android-apk-reverse-skills/commits)

</div>

---

## 这是什么

三个 **AI Agent 技能（Skill）**，处理 Android APK 二次改造里风险最高的三类操作：

| 想做的事 | 常见做法（风险高） | 这里的做法 |
|---|---|---|
| 改 APK 里的一句英文 | 反编译 → 改 smali → 重打包 → 重签名 | **原地等长改写**：文件长度与全部偏移零变化 |
| 去掉渠道 SDK 的实名认证弹窗 | 删代码 / 删权限（极易崩） | **改写 AXML 字符串池**，切断 SDK 的激活入口 |
| 验证改完的包能不能跑 | 手动装、手动点 | 模拟器自动化；跑不起来时用**静态验证**替代 |

**共同原则**：能不重打包就不重打包，能不动字节码就不动字节码。
文件长度一变，全部偏移和签名都要重算，出错面呈指数放大。

## 三个技能

### 1. `android-dex-string-patch` — dex 字符串原地替换

在已打包的 APK 里改写 `classes.dex` 的字符串（典型场景：把启动 Toast 换成自己的署名 / 中文文案），
**不重新编译、不新增或移动任何字节码指令、不改变文件长度**。

做法是挑一条可牺牲的**等长槽位**、原地重写内容、只改 `const-string` 指令的 2 字节操作数，
再重算签名与校验和。全程不碰指令流。

> **两条会直接导致"点击图标即闪退"的硬约束**（技能里写清了，这里先给结论）：
> ① `utf16_size` 是 **UTF-16 字符数**，不是 UTF-8 字节数 —— 纯 ASCII 串上二者恰好相等，
> 所以写成字节数在英文串上完全看不出问题，**一换中文就启动即崩**；
> ② 字符串索引表必须保持**字典序升序**，而中文码位大于全部 ASCII —— 所以中文串**不能随便挑槽位**。

### 2. `apk-channel-sdk-strip` — 渠道 SDK 剥离

渠道包（百分网 / 应用宝 / 九游等）会在原包上二次打包注入实名认证、广告、统计模块，
启动就弹窗。本技能通过**原地等长改写 `AndroidManifest.xml` 的 AXML 字符串池**，
切断渠道 SDK 的 `Application` / `Provider` 激活入口 —— 不重打包、不改字节码。

### 3. `android-emulator-autotest-setup` — 模拟器自动化验证

在 Windows 上搭 Android 模拟器 + 全自动"安装 / 启动 / 崩溃检测"流水线，
用于**无人值守地验证一个 APK 能否正常运行**。

包含**资源可行性预检**（内存 / 磁盘 / 虚拟化 / 硬件加速）——
很多环境根本跑不起模拟器，先花两分钟做预检，比折腾一小时才发现跑不起来划算。
并给出**模拟器不可用时的静态验证替代方案**（解析 APK 结构、校验偏移、检查字符串）。

## 安装

```bash
git clone https://github.com/lueuru/android-apk-reverse-skills.git
cp -r android-apk-reverse-skills/skills/* ~/.workbuddy/skills/
```

只想要一个也行 —— 每个 `skills/<名字>/` 都是自包含的：

```bash
cp -r android-apk-reverse-skills/skills/android-dex-string-patch ~/.workbuddy/skills/
```

## 使用示例

**例 1：给汉化包加上自己的署名弹窗**

> 「我想让游戏启动时弹一句『汉化版 by 我』，但不想重新打包。」

技能会定位进游戏链路上的 Toast 调用点，挑一条可牺牲的等长槽位改写，
并校验 `utf16_size` 与字典序 —— 这两条错一条就是启动闪退。

**例 2：进游戏弹实名认证**

> 「这游戏一进去就弹实名认证，是渠道包加的，怎么去掉？」

技能会定位 AXML 里渠道 SDK 的激活入口并等长改写。

**例 3：改完的包到底能不能跑**

> 「我改完怕有问题，但没有设备测试。」

技能先做资源预检；能起模拟器就走自动化安装+启动+崩溃检测，不能起就走静态验证。

## 为什么值得用

这几条是**在真实的 APK 改造项目上踩出来的**，不是从文档抄的：

- **"改成字节数"这个 bug 在英文串上完全看不出来** —— 只有换成中文才崩，而那时你已经改完准备发布了。
- **等长替换 = 零风险**：长度不变 → 偏移不变 → 校验和重算范围就那两处，出错面极小。
- **删代码 ≠ 去掉功能**：渠道 SDK 的激活是"入口驱动"的，切断入口比删代码安全得多。
- **先做资源预检**：模拟器跑不起来的原因往往是虚拟化没开或内存不足，与 APK 本身无关。

## 仓库结构

```
android-apk-reverse-skills/
├── README.md
├── LICENSE
├── .gitignore
└── skills/
    ├── android-dex-string-patch/
    │   └── SKILL.md
    ├── apk-channel-sdk-strip/
    │   └── SKILL.md
    └── android-emulator-autotest-setup/
        └── SKILL.md
```

## 适用与不适用

**适用**：已打包 APK 的轻量级改造 —— 文本替换、署名注入、渠道 SDK 剥离、无人值守验证。

**不适用**：
- 需要**改动逻辑**（改代码逻辑请走重编译路线，原地改写做不到）
- 加壳 / 强校验的应用（先脱壳）
- iOS（完全不同的一套）

## 合规提醒

这些技能用于**你自己拥有或已获授权修改**的应用。
去掉渠道 SDK 的弹窗、替换自己的署名，都应在你有权修改该应用的前提下进行；
**不要用它去篡改他人应用后分发**。

## 许可证

[MIT](./LICENSE) —— 自由使用、修改、再分发，保留版权声明即可。

