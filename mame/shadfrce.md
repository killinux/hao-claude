现在写总结文档：

---

# Shadow Force (shadfrce) 浏览器模拟器配置总结

## ROM 匹配分析

`shadfrce.zip` 共 15 个文件，对比 `shadfrce.cpp` 中三个驱动版本：

| 驱动 | 版本 | ROM 匹配 |
|------|------|----------|
| `shadfrce` | World Version 3 | ❌ 缺 5 个文件 |
| **`shadfrceu`** | **US Version 2** | **✅ 15/15 完全匹配** |
| `shadfrcej` | Japan Version 2 | ❌ 缺 5 个文件 |

**`shadfrce.cpp` 无需修改**，ZIP 里的 ROM 与 `shadfrceu` 驱动完全一致，CRC 全部吻合。

---

## shadfrce.zip 文件清单

| 文件 | 大小 | 用途 |
|------|------|------|
| `32a12-01.34` | 256KB | 主 CPU ROM（US 版） |
| `32a13-01.26` | 256KB | 主 CPU ROM（US 版） |
| `32a14-0.33` | 256KB | 主 CPU ROM（US 版） |
| `32a15-0.14` | 256KB | 主 CPU ROM（US 版） |
| `32a11-0.55` | 128KB | 字符图形（US 版） |
| `32j10-0.42` | 64KB | 音效 CPU ROM |
| `32j1-0.4` ~ `32j3-0.6` | 各 1MB | 背景图块 |
| `32j4-0.12` ~ `32j8-0.32` | 各 2MB | 精灵图形 |
| `32j9-0.76` | 512KB | OKI PCM 采样 |

---

## 修改的文件

### `/opt/workspace/emularity/shadfrce.html`

| 项目 | 修改前 | 修改后 |
|------|--------|--------|
| `driver` | `"shadfrce"` (World，缺 ROM) | **`"shadfrceu"`** (US，完全匹配) |
| `nativeResolution` | `256, 256`（错误） | **`320, 240`**（来自 cpp 中 `set_raw` 参数） |
| `emulatorJS` | `"emulators/mame/mameshadfrce.js"`（错误路径） | **`"emulators/mame/shadfrce.js"`** |
| `title` | `example arcade game` | `Shadow Force` |

---

## 编译命令（macOS）

```bash
cd /opt/workspace/mame
emmake make SUBTARGET=shadfrce SOURCES=src/mame/technos/shadfrce.cpp REGENIE=1 -j$(sysctl -n hw.logicalcpu)
```

编译产物：
- `/opt/workspace/mame/shadfrce.js`
- `/opt/workspace/mame/shadfrce.wasm`

（符号链接 `emularity/emulators/mame → /opt/workspace/mame` 已存在，无需额外操作）

---

## 关于父子 ROM 集

`shadfrceu` 是 `shadfrce` 的 clone（子集），MAME 加载时会依次搜索：
1. `shadfrceu.zip`（不存在）
2. `shadfrce.zip`（父集，即我们的 zip）✅

因此 zip 文件名保持 `shadfrce.zip` 不变即可正常使用。
