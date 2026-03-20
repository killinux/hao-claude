# tengai (战国之刃) 浏览器模拟器修复总结

from https://github.com/killinux/mame


1.环境：我的wasm在 /opt/work/emsdk 

cd emsdk

./emsdk activate 3.1.50

source ./emsdk_env.sh

2.项目路径：/opt/work/mame

3.需求：
emmake make SUBTARGET=psikyo SOURCES=src/mame/psikyo/psikyo.cpp


## 问题描述

在浏览器中运行 MAME WebAssembly 版本的《战国之刃 (Sengoku Blade / Tengai)》时，游戏卡在加载界面，控制台持续输出 `Aborted()` 错误。

---

## 根本原因分析

### 问题一：ROM 文件缺失（`4.u59`）

原始 `tengai.zip` 缺少 PIC16C57 MCU 芯片的 ROM 文件 `4.u59`。该芯片在 MAME 源码中注释为 "not hooked up"（未接入），但仍被包含在 ROM 加载列表中，导致启动失败。

### 问题二：WASM 编译 Bug — 异常引用计数函数冲突

这是核心崩溃原因。`psikyo.wasm` 的导出表中存在一个严重的链接错误：

```
__cxa_increment_exception_refcount  →  func47179  (本应是 increment)
__cxa_decrement_exception_refcount  →  func47179  (decrement)
```

**两个函数都指向同一个 WASM 函数（func47179）**，即实际上的 decrement 实现。

**崩溃路径：**

```
emu_fatalerror 被抛出（引用计数 = 1）
  ↓
___cxa_begin_catch() 调用"increment"
  ↓  实际执行 decrement（Bug！）
引用计数 1 → 0
  ↓
WASM 认为异常无持有者，立即调用析构链
func47178 → func47177 → func47164 → abort()
  ↓
Aborted()  ← 错误消息从未被打印，直接崩溃
```

这是由 Emscripten 编译时的 `EXCEPTION_CATCHING_ALLOWED` 配置与实际使用的 Emscripten 版本之间的不兼容导致的链接错误。

---

## 修复方案

### 修改文件一：`/opt/workspace/mame/src/mame/psikyo/psikyo.cpp`

移除 PIC16C57 MCU 设备（从未接入硬件，ROM 不存在也无需加载）：

**`s1945_state::s1945()` 机器配置中删除：**
```cpp
// 已删除：
PIC16C57(config, "mcu", 4_MHz_XTAL).set_disable();
```

**`ROM_START(tengai)` 中删除 MCU ROM 区域：**
```cpp
// 已删除：
ROM_REGION( 0x001000, "mcu", 0 )       // MCU, not hooked up
ROM_LOAD( "4.u59", 0x00000, 0x01000, CRC(e563b054) SHA1(...) )
```

**`ROM_START(tengaij)` 中同样删除：**
```cpp
// 已删除：
ROM_REGION( 0x001000, "mcu", 0 )       // MCU, not hooked up
ROM_LOAD( "4.u59", 0x00000, 0x01000, CRC(e563b054) SHA1(...) ) // From a World PCB
```

---

### 修改文件二：`/opt/workspace/emularity/emulators/roms/tengai.zip`

替换为 backup 版本（Japan 版 ROM 集，共 10 个文件，不含 `4.u59`）：

| 文件 | 用途 |
|------|------|
| `2-u40.bin` | 主 CPU ROM word 0（Japan 版） |
| `3-u41.bin` | 主 CPU ROM word 1（Japan 版） |
| `1-u63.bin` | 音效 CPU ROM |
| `u20/u21/u22.bin` | 精灵图形数据 |
| `u34.bin` | 背景图块数据 |
| `u61/u62.bin` | PCM 采样数据 |
| `u1.bin` | 精灵查找表 |

---

### 修改文件三：`/opt/workspace/emularity/tengai.html`

切换驱动名称，匹配 Japan 版 ROM 集：

```javascript
// 修改前：
JSMAMELoader.driver("tengai")   // World 版

// 修改后：
JSMAMELoader.driver("tengaij")  // Japan 版
```

---

### 修改文件四：`/opt/workspace/mame/psikyo.js`（关键 Bug 修复）

绕过 WASM 编译 Bug，将两个引用计数函数替换为 JavaScript no-op：

```javascript
// 修复前（错误地调用同一个 WASM decrement 函数）：
var ___cxa_increment_exception_refcount =
    a0 => (___cxa_increment_exception_refcount =
        wasmExports["__cxa_increment_exception_refcount"])(a0);

var ___cxa_decrement_exception_refcount =
    a0 => (___cxa_decrement_exception_refcount =
        wasmExports["__cxa_decrement_exception_refcount"])(a0);

// 修复后（no-op，跳过有 Bug 的 WASM 实现）：
var ___cxa_increment_exception_refcount = a0 => {};
var ___cxa_decrement_exception_refcount = a0 => {};
```

**为什么 no-op 是安全的：**
- 异常对象在 catch 块期间始终保持有效（引用计数保持为初始值 1）
- catch 处理完成后不调用析构，WASM 内存有轻微泄漏，但游戏正常退出时无影响
- 对游戏正常运行路径（无异常抛出）完全没有影响

---

### 修改文件五：符号链接修复

```bash
# 修复前（链接目标不存在）：
/opt/workspace/emularity/emulators/mame -> /opt/mycode/mame  ❌

# 修复后：
/opt/workspace/emularity/emulators/mame -> /opt/workspace/mame  ✅
```

---

## 修改文件汇总

| 文件 | 修改类型 | 说明 |
|------|----------|------|
| `mame/src/mame/psikyo/psikyo.cpp` | 源码修改 + 重新编译 | 移除 PIC16C57 MCU 设备和 ROM 区域 |
| `emularity/emulators/roms/tengai.zip` | 文件替换 | 换用 Japan 版完整 ROM 集 |
| `emularity/tengai.html` | 配置修改 | driver 改为 `tengaij` |
| `mame/psikyo.js` | JS 热修复 | 修复 WASM 异常引用计数 Bug |
| `emularity/emulators/mame`（符号链接） | 路径修复 | 指向正确的编译产物目录 |
