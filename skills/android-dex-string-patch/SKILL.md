---
name: android-dex-string-patch
description: 在已打包的 Android APK 内原地改写 classes.dex 里的字符串（典型场景：把启动 Toast / 提示语换成汉化署名或中文文案），不重新编译、不新增/移动任何字节码指令、不改变文件长度。适用于 Unity/原生 APK 的轻量级文本替换、署名注入、提示语本地化。当任务涉及「改 APK 里的某句英文/加署名弹窗/dex 字符串替换」时使用。
agent_created: true
---

# Android DEX 字符串原地打补丁

## 何时用

- 要把 APK 里某条**硬编码英文字符串**换成中文/署名（Toast、错误提示、日志横幅）。
- 手头只有成品 APK，没有源码，也不想重建工程。
- 要求**最大兼容性**：不改字节码、不改文件大小、不改任何 offset。

不适用：需要新增字符串常量、需要改逻辑分支、需要新增类 —— 那些得改 smali 重编译。

---

## DEX 结构速查（只列本任务要碰的）

```
header (0x70 字节)
  0x00 magic   "dex\n035\0"
  0x08 checksum  u32 LE  adler32(dex[12:])
  0x0C signature 20×u8   SHA-1(dex[32:])
  0x20 file_size u32 LE  ← 等于文件实际长度
  0x38 string_ids_size u32 │ 0x3C string_ids_off u32   ← 注意：这里读到的是 size,off 连读
  0x40 type_ids_size   u32 │ 0x44 type_ids_off   u32
  ...
string_ids[]  每项 4 字节 u32 offset，指向 string_data_item
string_data_item {
    uleb128 utf16_size;   ★★ 单位是 UTF-16 code unit 个数，不是字节数！
    ubyte[] data;         MUTF-8 编码（非标准 UTF-8）
    ubyte   0x00;         终止符
}
code_item {
    registers u16, ins u16, outs u16, tries u16,
    debug_info u32, insns_size u32, insns[insns_size] u16
    // 指令从 code_off + 16 开始，每 unit 2 字节
}
const-string = 指令格式 21，占 2 个 unit：
    unit0 = opcode(0x1a) | (vAA << 8)
    unit1 = string_ids 索引（u16 LE）
```

---

## ★★ 两个致命坑（踩过，直接导致「点进去就闪退」）

### 坑 1：`utf16_size` 写成字节数

```python
# ✗ 错（ASCII 串上恰好正确，含中文必崩）
blob = bytes([len(text.encode('utf-8'))]) + text.encode('utf-8') + b'\x00'

# ✓ 对
def utf16_len(s):
    return len(s.encode('utf-16-le')) // 2      # 含超 BMP 字符也正确
```

后果：ART 的 `dex_file_verifier.cc::CheckIntraStringDataItem` 判定
`utf16_size != 实际 MUTF-8 解出的 UTF-16 长度` → **VerifyError → 进程启动即崩**。

**为什么容易漏**：原串是纯 ASCII 时 `字节数 == 字符数`，错写法"侥幸通过"原版自洽检查。
一旦替换成含中文的串就必然暴露。**这是最危险的假阳性安全区。**

### 坑 2：破坏 `string_ids` 字典序

DEX 规范：`string_ids` 必须按字符串内容的 **UTF-16 code point 升序**排列。

**中文（CJK）的 code point 大于所有 ASCII**（`'汉'`=U+6C49，`'z'`=0x7A），
所以一个含中文的串按序必须排在**所有 ASCII 串之后**。
如果随便挑一个中间槽位塞中文 → 与该槽位之后的 ASCII 串逆序。

**解法：挑槽位时看紧邻的前驱/后继首字符，利用「字典序在首字符即分出胜负」**

给定槽位 `i`，其合法区间是 `(str[i-1], str[i+1])`。只要新串满足：

```
str[i-1] < 新串 < str[i+1]
```

即天然落位正确，中文放在串内任意位置都不影响（首字符已经定胜负）。

实战例子：`#554='YWRkUmVxdWVzdFByb3BlcnR5='`，`#556='Z'`
→ 新区间接受任意 `'Y' + 第2字符 > 'W'` 的串，包括 `"You are playing 汉化版..."`。
（`'o'`=0x6F > `'W'`=0x57，且首字符 `'Y'` < `'Z'` ✓）

**槽位选择检查清单**：
1. 原内容在正常流程里**不会显示**（挑那种只走错误分支的冗长提示最安全）。
2. 原内容足够长（能装下新串的 MUTF-8 字节数）。
3. 前驱/后继区间能容纳新串。**优先让首字符与原串相同** —— 这样区间必然不变。
4. 原始 `utf16_size` 自身必须自洽（不改动时能通过校验）。

---

## 等长替换法（本方案的核心价值：零 offset 漂移）

目标：保持 `string_data_item` 的**总占用字节数**不变。

```
原占用 = uleb_size(原 utf16_size) + len(原 MUTF-8) + 1
新占用 = uleb_size(新 utf16_size) + len(新 MUTF-8) + 1
要求 新占用 == 原占用
```

- 若 `新占用 > 原占用` → 放不下，换槽位或缩短文本。
- 若 `新占用 < 原占用` → 用 `0x00` 补齐（多数 verifier 只按 `string_ids` 的 offset 跳转读取，
  不扫描 string_data 区要求紧排；但**能等长就千万别补** —— 零 padding 才是绝对安全）。
- **最优做法**：直接设计文本长度，让 `len(新 MUTF-8) == len(原 MUTF-8)`（ASCII 原串则 `== 原字符数`）。

等长的收益：**文件大小不变、所有 string_ids offset 不变、type/field/method/class 区完全不动** ——
唯一改动就只剩"那段数据 + const-string 的 2 字节索引"，风险面最小。

---

## 完整流程

```python
import struct, zlib, hashlib

def read_uleb(d, p):
    r = 0; s = 0
    while True:
        b = d[p]; p += 1
        r |= (b & 0x7F) << s; s += 7
        if not (b & 0x80): break
    return r, p

def uleb_size(v):
    n = 1
    while v >= 0x80: v >>= 7; n += 1
    return n

def mutf8_encode(s):
    """MUTF-8：与 UTF-8 仅在 U+0000(C0 80) 和超 BMP(CESU-8 代理对) 上不同。
    BMP 内普通字符（含全部汉字）编码与 UTF-8 完全相同。"""
    out = bytearray()
    for ch in s:
        cp = ord(ch)
        if cp == 0:
            out += b'\xC0\x80'
        elif cp < 0x80:
            out.append(cp)
        elif cp < 0x800:
            out.append(0xC0 | (cp >> 6)); out.append(0x80 | (cp & 0x3F))
        elif cp < 0x10000:
            out.append(0xE0 | (cp >> 12))
            out.append(0x80 | ((cp >> 6) & 0x3F))
            out.append(0x80 | (cp & 0x3F))
        else:
            cp -= 0x10000
            for u in (0xD800 + (cp >> 10), 0xDC00 + (cp & 0x3FF)):
                out.append(0xE0 | (u >> 12))
                out.append(0x80 | ((u >> 6) & 0x3F))
                out.append(0x80 | (u & 0x3F))
    return bytes(out)

def utf16_len(s):
    return len(s.encode('utf-16-le')) // 2

def read_item(d, off):
    """返回 (utf16_size, raw, total_bytes)"""
    decl, p = read_uleb(d, off)
    q = p
    while q < len(d) and d[q] != 0: q += 1
    return decl, bytes(d[p:q]), (q + 1) - off

# ---- 1. 读 dex ----
d = bytearray(open("classes.dex", "rb").read())
assert d[:8] == b'dex\n035\x00'
n_str, str_off = struct.unpack_from('<II', d, 56)

all_s = []
for i in range(n_str):
    o = struct.unpack_from('<I', d, str_off + 4 * i)[0]
    all_s.append(read_item(d, o)[1].decode('utf-8', 'replace'))

# ---- 2. 定位槽位并校验 ----
SLOT = 555
slot_off = struct.unpack_from('<I', d, str_off + 4 * SLOT)[0]
old16, old_raw, old_total = read_item(d, slot_off)

new_text = "You are playing 汉化版 by uruban | QQ群：957336473"
nb = mutf8_encode(new_text)
n16 = utf16_len(new_text)
new_total = uleb_size(n16) + len(nb) + 1
assert new_total <= old_total, "放不下"
pad = old_total - new_total

# ---- 3. 排序校验（只查槽位左右各 2 条） ----
probe = all_s[:]; probe[SLOT] = new_text
for i in range(max(1, SLOT - 2), min(n_str, SLOT + 2)):
    assert probe[i - 1] <= probe[i], "破坏 string_ids 排序 @%d" % i

# ---- 4. 原地重写（等长） ----
payload = bytes([n16]) + nb + b'\x00' + b'\x00' * pad
assert len(payload) == old_total
d[slot_off:slot_off + old_total] = payload

# ---- 5. 改 const-string 操作数 ----
CODE_OFF = 86148; INSN = 16
base = CODE_OFF + 16
w0 = struct.unpack_from('<H', d, base + 2 * INSN)[0]
assert (w0 & 0xFF) == 0x1A, "不是 const-string"
struct.pack_into('<H', d, base + 2 * INSN + 2, SLOT)

# ---- 6. 重算双哈希（顺序不能反） ----
d[12:32] = hashlib.sha1(bytes(d[32:])).digest()
struct.pack_into('<I', d, 8, zlib.adler32(bytes(d[12:])) & 0xFFFFFFFF)
```

---

## 交付前必过的校验（缺一不可）

| # | 检查 | 判据 |
|---|---|---|
| 1 | magic | `dex\n035\0` |
| 2 | file_size(@0x20) == len(dex) | 相等 |
| 3 | **每条** string_data_item 的 `utf16_size` == MUTF-8 实解 UTF-16 长度 | 不符 **0** 处 |
| 4 | **每条**字符串以 `0x00` 结尾 | 缺 **0** 处 |
| 5 | `string_ids` 相邻逆序数 | **0** 处 |
| 6 | signature == SHA-1(dex[32:]) | 相等 |
| 7 | checksum == adler32(dex[12:]) | 相等 |
| 8 | const-string 索引指向目标串 | 读回即新文本 |
| 9 | 文件长度与原 dex 相同 | 相等（等长法下必然） |

**第 3、5 项是本次事故的直接原因，必须逐条全表扫，不能用抽样。**

### 独立交叉验证（避免"自己验自己"）
用**不同实现**再验一遍：
- **LIEF**（`pip install lief`）：`lief.DEX.parse(p)` → `len(list(d.strings))`、`d.classes`、`d.methods`，
  以及 `d.header` 的 Map/Data offset 是否有效。能完整解出类/方法即说明结构合法。
- `androguard`、`apktool`/`baksmali` 反编译（若能跑通即通过）。

---

## APK 重组注意（若需替换回包）

- 保持每条目的 `compress_type`：`resources.arsc` / `AndroidManifest.xml` 若原为 **STORED 必须保持 STORED**，
  且 `resources.arsc` 需 **4 字节对齐**。
- **跳过 `META-INF/`**（旧签名不能带进新包）。
- split 分卷（`xxx.split0..N`）只写 `.split0`，其余跳过，否则同一逻辑文件写 N 份。
- 不写空目录条目。
- 签完务必核验：`zipalign verified` + `signature verified [v1, v2, v3]`。

---

## 反例与教训

- **别用"字节数"当 `utf16_size`**，即使原版是 ASCII 也不要用 —— 一旦换中文就崩。
- **别在没查前驱/后继的情况下挑槽位放中文**。
- **别只看"文件长度没变"就认为安全** —— 长度没变可以掩盖"内容位置错、字段单位错"。
- **别用单一样本（ASCII 原串）验证编码实现**，样本必须含非 ASCII。
- 改完 dex **必须重算两个哈希**，且 `signature` 要在 `checksum` 之前算（checksum 覆盖 signature 字段）。
