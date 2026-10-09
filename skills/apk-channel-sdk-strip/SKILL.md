---
name: apk-channel-sdk-strip
description: 从已打包的 Android APK 里剥离渠道 SDK（百分网/应用宝/九游等二次打包注入的实名认证、广告、统计模块），使其不在启动时弹窗。做法是通过原地等长改写 AndroidManifest.xml 的 AXML 字符串池，切断渠道 SDK 的 Application/Provider 激活入口，零字节码改动。当任务涉及「进游戏弹实名认证/渠道弹窗/要求删除渠道 SDK/APK 被二次打包」时使用。
agent_created: true
---

# APK 渠道 SDK 剥离

## 何时用

- 游戏/应用一启动就弹「实名认证」「渠道登录」「健康游戏忠告」，而**原版并无此行为**。
- 用户要求「把 XX 网的东西删掉」。
- APK 是从第三方下载站拿到的，包名正常但行为异常。

## 第一步：三处取证（不要凭名字猜）

渠道 SDK 通常在三处留痕，**三处都要查**：

### ① dex 里的类包名

```python
import lief, zipfile
for name in [n for n in z.namelist() if n.endswith('.dex')]:
    d = lief.DEX.parse(dex_path)
    from collections import Counter
    pk = Counter()
    for c in d.classes:
        p = c.fullname.strip('L').replace('/', '.').rstrip(';')
        pk['.'.join(p.split('.')[:3])] += 1
```
看有没有与游戏无关的包，如：
- `com.byfen.authentication`（百分网，实测 47 个类）
- `com.qbsu.wem`、`com.zdwrt.fmxvl`（随机字符 = 混淆的加固/广告 SDK）
- 应用宝 `com.tencent.tmgp`、九游 `com.u8.sdk`、小米 `com.xiaomi.gamecenter.sdk` 等

### ② assets 下的渠道目录

```python
[n for n in z.namelist() if n.startswith('assets/') and not n.startswith('assets/bin/')]
```
实测百分网会投 `assets/byfen/authentication/bf_layout_dialog_authentication.xml` 等 **7 个认证弹窗布局**。

### ③ manifest 的激活入口（**关键**）

```python
from pyaxmlparser import APK
from lxml import etree
x = etree.tostring(APK(apk).get_android_manifest_xml()).decode()
# 找 application 的 android:name、以及 <queries>/<provider>/<service> 里的渠道包名
```
```xml
<application android:name="com.byfen.authentication.MyApp" ...>
<queries><package android:name="com.byfen.market"/></queries>
```

**触发链**：进程启动 → 系统实例化 manifest 指定的 Application 类 → `MyApp.onCreate()`
→ 注册 `ActivityLifecycleCallbacks` → 首个 Activity 起来时检查认证状态 → 弹实名认证。

**所以切断 `android:name` 就能让整条链断掉**，不需要动 dex 里那 47 个类。

---

## 第二步：AXML 原地等长改写

### ★★ AXML 字符串池格式（别按 DEX 的经验套）

```
XML chunk header @0 : type(2)=0x0003  headerSize(2)  size(4)
StringPool chunk @8: type(2)=0x0001  headerSize(2)=0x1C  size(4)
    stringCount(4) @8+8   styleCount(4) @+12   flags(4) @+16
    stringsStart(4) @+20  stylesStart(4) @+24
    offsets[stringCount] (u32 each) @+28      ← 相对 stringsStart 的偏移
    strings data @ chunk + stringsStart
```

**每条字符串**：
- `flags & 0x100`（UTF-8 池）：`[uleb128 utf16_len][uleb128 byte_len][数据][0x00]`
- **否则（UTF-16 池）：`[u16 LE utf16_len][UTF-16LE 数据][0x0000]`** ← **是 u16，不是 uleb128！**

> 实测验证：AndroidManifest 第一个字符串 `compileSdkVersion`，
> `d[272:274] = 11 00`（u16 = 17 ✓）、`d[274] = 0x63`（'c' ✓）。
> 若按 uleb128 解析会**整体错位 1 字节**，读出 `挀漀洀瀀椀氀攀` 这种乱码。

### 定位字符串：直接搜字节序列 + 双重校验（最稳）

```python
def find_utf16_string(d, s):
    pat = s.encode('utf-16-le')
    out, i = [], 0
    while True:
        j = d.find(pat, i)
        if j < 0:
            break
        ok_pre  = j >= 2 and struct.unpack_from('<H', d, j-2)[0] == len(s)   # 前 2 字节 = 长度
        ok_post = bytes(d[j+len(pat): j+len(pat)+2]) == b'\x00\x00'         # 后 2 字节 = NUL
        if ok_pre and ok_post:
            out.append((j, j-2, 2 + len(pat) + 2))     # (数据起点, 前缀起点, 总占用)
        i = j + 1
    return out
```
双重校验可排除「该串是某个更长串前缀」的误命中（如 `com.byfen.market` vs `com.byfen.market.provider`）。

### 改写：新串更短 → 原占用内写「新串 + 0x00 填充」

```python
new_blob = struct.pack('<H', len(s_new)) + s_new.encode('utf-16-le') + b'\x00\x00'
pad = (2 + len(s_old)*2 + 2) - len(new_blob)
d[start: start + len(new_blob) + pad] = new_blob + b'\x00' * pad
```

收益：**AXML 长度不变、StringPool 各 offset 不变、后续 chunk 全部不动**。
结尾写 `0x00` 填充，解析器读完 u16 长度 → 读字符 → 读到 NUL 即正常结束，尾部填充无人访问。

### 改成什么

| 原值 | 建议改成 | 理由 |
|---|---|---|
| `android:name="渠道类"` | `android.app.Application` | 系统自带类，语义等价于「不指定」；原生 Unity 应用无需自定义 Application |
| `<queries>` 里的渠道包名 | `android` | 只影响包可见性，无功能影响 |

**不要**改成不存在的类名 —— 会 `ClassNotFoundException` 直接崩。

---

## 第三步：剔除渠道资源

重打包时按前缀跳过，例如 `assets/byfen/`。同时 `META-INF/` 必须跳过（旧签名）。

---

## 验证清单

| # | 检查 | 判据 |
|---|---|---|
| 1 | 改写后 AXML 长度 | 与原始**完全相等** |
| 2 | 第三方解析器复核 | `pyaxmlparser` 解出的 `android:name` == 新值 |
| 3 | 全文关键词 | 解析后的 XML 里渠道关键字为 0 |
| 4 | 包内残留 | `assets/<渠道>/` 条目数 == 0 |
| 5 | 签名 | zipalign verified + signature verified v1/v2/v3 |

---

## 反例与教训

- **别只删 dex 里的类** —— 字节码级删除极易破坏 `class_defs`/`method_ids` 索引；而切断 manifest 入口
  只需改 2 个字符串，风险低几个数量级，效果相同。
- **别按 DEX 经验解析 AXML** —— DEX 的 UTF-16 字符串前缀是 uleb128，**AXML 是 u16**。
- **别用「找第一个 `xx 00 xx 00`」式的启发式定位** —— 一定带长度前缀与 NUL 后缀双重校验。
- **别改完不校验长度** —— 长度变了就说明不是「原地等长」，offset 可能已被破坏。
- 改完**必须重新签名**，且要过 zipalign（`resources.arsc` 需 STORED + 4 字节对齐）。
