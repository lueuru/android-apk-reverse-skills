---
name: android-emulator-autotest-setup
description: 在 Windows 上搭建 Android 模拟器 + 全自动 APK 安装/启动/崩溃检测流水线，用于「无人值守地验证一个 APK 能否正常运行」。包含资源可行性预检（内存/磁盘/虚拟化/硬件加速）、SDK 与 AVD 搭建、以及模拟器不可用时的静态验证替代方案。当任务涉及「自动测试 APK」「模拟器跑游戏」「无需人工参与的安装验证」时使用。
agent_created: true
---

# Android 模拟器自动化测试环境搭建

## 第 0 步（最重要）：先做资源可行性预检

**别急着下载 SDK**。模拟器对资源要求很硬，先花 1 分钟查清，能省下几 GB 下载和几十分钟试错。

```powershell
# 内存与磁盘
$cs = Get-CimInstance Win32_ComputerSystem
"内存(GB): $([math]::Round($cs.TotalPhysicalMemory/1GB,1))"
"机器型号: $($cs.Model)"          # 若是 *Hypervisor* / *Virtual Machine* -> 本机是虚拟机
"HypervisorPresent: $($cs.HypervisorPresent)"
# CPU 虚拟化标志
$p = Get-CimInstance Win32_Processor
"VMMonitorModeExtensions: $($p.VMMonitorModeExtensions)"
"SecondLevelAddressTranslationExtensions: $($p.SecondLevelAddressTranslationExtensions)"
```

**判据表**：

| 检查 | 通过线 | 不通过的后果 |
|---|---|---|
| 物理内存 | **≥ 8 GB**（4 GB 绝对不够，可用内存常只剩 0.6 GB）| 启动瞬间 OOM |
| 可用磁盘 | **≥ 30 GB** | SDK(5 GB)+系统镜像(最大 3.3 GB)+AVD 数据放不下 |
| 机器不是虚拟机 | 是物理机最稳 | 虚拟机需**嵌套虚拟化**，且加速驱动常装不上 |
| `VMMonitorModeExtensions=True` | 必须 | 无 VT-x 只能纯软模拟（极慢） |
| 硬件加速驱动可安装 | 必须 | 见下方「三大拦路虎」 |

> ★ **`x86_64` 系统镜像实际约 3.3 GB**（不是常见的"几百 MB"印象）。
> `armeabi-v7a` 镜像稍小，但纯软件模拟 ARM 跑 Unity 游戏基本不可行。

## 第 1 步：装 SDK（用已有 JRE 即可，不必装完整 JDK）

```bash
SDK="C:/android-sdk"
curl -L -o clt.zip "https://dl.google.com/android/repository/commandlinetools-win-11076708_latest.zip"
mkdir -p "$SDK/cmdline-tools" && unzip -q clt.zip -d "$SDK/cmdline-tools"
mv "$SDK/cmdline-tools/cmdline-tools" "$SDK/cmdline-tools/latest"

export JAVA_HOME="<任意 JRE 8+ 目录>"       # 复用签名用的 JRE 即可
export ANDROID_HOME="$SDK" ANDROID_SDK_ROOT="$SDK"
cd "$SDK/cmdline-tools/latest/bin"
yes | cmd //c "sdkmanager.bat --licenses"
cmd //c "sdkmanager.bat platform-tools emulator \"system-images;android-22;default;x86_64\""
```

**注意**：
- `sdkmanager` 由 **JRE** 驱动即可，不需要 JDK。
- 从 Bash 调 `.bat` 时必须 `cmd //c "xxx.bat"`；**但引号会被剥掉**，
  含分号的参数（如 `system-images;android-22;...`）会解析失败。
  → **改用 Python `subprocess.run([bat, ...])` 直接传参数列表**（不经 shell）。

## 第 2 步：建 AVD（同样用 Python 调，避开引号问题）

```python
import subprocess, os
env = dict(os.environ); env.update({"JAVA_HOME": "...", "ANDROID_HOME": "...",
         "ANDROID_SDK_ROOT": "...", "ANDROID_AVD_HOME": "C:/android-sdk/avd"})
subprocess.run(["C:/android-sdk/cmdline-tools/latest/bin/avdmanager.bat",
                "create","avd","-n","af_test","-k","system-images;android-22;default;x86_64",
                "-d","Nexus 5","--force"], input=b"no\n", env=env)
```

创建后改 `config.ini` 降配（低内存机器必做）：
```ini
hw.ramSize=1536
hw.cpu.ncore=2
hw.gpu.enabled=yes
hw.gpu.mode=swiftshader_indirect
disk.dataPartition.size=2048M
hw.audioInput=no
hw.audioOutput=no
hw.camera.back=none
hw.camera.front=none
```

启动（无窗口 + 软件渲染）：
```bash
emulator -avd af_test -no-window -no-audio -no-boot-anim \
         -gpu swiftshader_indirect -no-snapshot -memory 1536
```

## ★★ 三大拦路虎（我全遇到了）

### 1. `x86_64 emulation currently requires hardware acceleration!`
x86 镜像**必须**有硬件加速。两条路：
- **AEHD**（Google 官方，`sdkmanager extras;google;Android_Emulator_Hypervisor_Driver`
  然后跑 `silent_install.bat`）→ **需要管理员提权**，非提权会话会「拒绝访问」。
- **WHPX**（`Enable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform`）
  → **需要重启**，且要求 Hyper-V 平台可用。

### 2. `sc.exe` 被安全策略黑名单阻断
`sc query` / `sc create` 会被沙箱直接拦，报
`PROGRAM BLOCKED BY SECURITY POLICY`，**不可绕过、不可换 shell**。
→ 这意味着**任何需要注册系统服务的驱动安装都做不了**，AEHD 因此装不上。
→ 用户需自行在「安全中心 → 命令安全 → 程序黑名单」移除。

### 3. 内存不足（4 GB 机器）
`可用内存 0.6 GB` 时启动模拟器必 OOM。**这是物理上限，改配置无解。**
→ 换 **ARM 镜像**也不行（纯软模拟跑游戏慢到不可用）。

## ★ 模拟器不可用时的替代：深度静态验证

**「自检通过 ≠ 能跑」**，但静态验证能覆盖**大部分真实失败模式**，值得认真做：

| 组 | 覆盖的失败模式 | 关键检查 |
|---|---|---|
| dex | 闪退 VerifyError | `utf16_size` 全表正确（**是 UTF-16 字符数不是字节数**）/ `string_ids` 升序 / 双哈希 / 用 LIEF 独立解析 |
| 原生库 | 闪退 架构不匹配 | 读 `.so` ELF 头 `e_machine`（0x28=ARM, 0x3=x86, 0xb7=ARM64, 0x3e=x86_64）与目录名比对 |
| 资源 | 闪退 结构破坏 | 对象表「末地址 == 文件大小」/ `pathID`/`typeID` **与原版逐项比对** |
| 字体 | 方块字 / 空白 | Tofu Check（已用字符 ⊆ 图集 char id）+ 图集非透明像素抽样 |
| 深度 | 内容偏移错位 | **用 UnityPy 读取全部对象，并与原版对比可读性** |
| 打包 | 安装失败 | v1/v2/v3 签名 / STORED / zip 完整性 / 关键条目 |

**两个判据陷阱**（都踩过）：
1. **「全部对象可读」是假阳性判据** —— MonoBehaviour 多数无 type tree、AudioClip 编码不支持，
   **原版就读不出**。必须写成「**与原版一致**」。
2. **`resources.arsc` 4 字节对齐**只在 `targetSdk >= 30` 时强制；
   `targetSdk=22` 的老包**原版本来就不对齐**，不能判为缺陷。

## ★ 留给「真机 / 加内存后」的自动化脚本骨架

判定逻辑**不能是「启动一下就算过」**，要逐项检测：

```python
crashed      = 'FATAL EXCEPTION' in log
anr          = 'ANR in' in log
verify_error = any(k in log for k in ('VerifyError','ClassNotFoundException',
                                      'NoClassDefFoundError','UnsatisfiedLinkError',
                                      'dlopen failed'))
died_after   = marks[0].pid and not marks[-1].pid        # 启动后进程消失
stuck_launch = all('launcher' in m.fg.lower() for m in marks[-3:])   # 疑似卡 loading
```
采样用 `pidof <pkg>` + `dumpsys activity activities` 取 `mResumedActivity`，
截图用 `adb exec-out screencap -p`（**注意是 `exec-out` 不是 `shell`**，否则 PNG 会被换行符弄坏）。

## 反例与教训

- **别跳过第 0 步的预检** —— 4 GB 内存的机器上装了 5.4 GB SDK 才发现跑不起来。
- **别从 Bash 直接传含引号的 `.bat` 参数** —— 用 Python `subprocess` 传列表。
- **临时文件放系统 Temp** —— 项目目录可能受安全删除守卫约束，`os.remove` 会抛
  `SAFE_DELETE_FAIL_CLOSED`（这是清理失败，不是验证失败，别误判）。
- **别把「UnityPy 读不出某些对象」当成损坏** —— 先和原版比。
- **`sc.exe` 被黑名单时不要尝试绕过**（换 shell / 写脚本都不行），直接换方案。
