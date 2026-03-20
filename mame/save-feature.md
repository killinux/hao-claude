# MAME WebAssembly 存档功能实现文档

## 功能概述

在游戏页面中增加手动导出/导入存档（Save State）的功能，让用户可以将 MAME 的存档文件下载到本地，并在需要时上传恢复。

---

## 技术原理

### MAME 存档机制

MAME 支持 Save State（即时存档），在游戏运行中：
- 按 `Shift + F7` → 存档到当前槽位
- 按 `F7` → 从当前槽位读档

存档文件保存在 MAME 的虚拟文件系统中，路径格式为：

```
/sta/<驱动名>/<槽位号>.sta
```

例如：`/sta/tengaij/0.sta`（槽位 1 的存档）

### Emscripten FS API

MAME WebAssembly 使用 Emscripten 的虚拟文件系统（MemFS + IDBFS）。通过全局 `Module.FS` 可以直接读写这个虚拟文件系统：

```javascript
// 读取文件（返回 Uint8Array）
var data = Module.FS.readFile('/sta/tengaij/0.sta');

// 写入文件
Module.FS.writeFile('/sta/tengaij/0.sta', uint8ArrayData);

// 检查目录是否存在
Module.FS.stat('/sta/tengaij/');

// 创建目录
Module.FS.mkdir('/sta/tengaij/', 0o777);
```

---

## 页面结构

### HTML UI（`tengai.html`）

```html
<div id="save-controls">
  槽位:
  <select id="save-slot">
    <option value="0">1</option>
    <option value="1">2</option>
    <option value="2">3</option>
  </select>
  <button onclick="exportSave()">📥 导出存档</button>
  <button onclick="document.getElementById('save-file-input').click()">📤 导入存档</button>
  <input type="file" id="save-file-input" accept=".sta" style="display:none" onchange="importSave(this)">
  <span id="save-status"></span>
</div>
<p>游戏内按 <b>Shift+F7</b> 存档 | <b>F7</b> 读档 | 存完再点"导出存档"备份到本地</p>
```

---

## 核心 JavaScript 实现

### 1. 路径查找（`findSavePath`）

不同编译版本的 MAME，存档目录路径可能不同，使用候选路径列表逐一探测：

```javascript
var GAME_NAME = 'tengaij';

var SAVE_BASE_PATHS = [
  '/sta/' + GAME_NAME + '/',
  '/home/web_user/.mame/sta/' + GAME_NAME + '/',
  '/mame/sta/' + GAME_NAME + '/',
];

function findSavePath(slot) {
  for (var i = 0; i < SAVE_BASE_PATHS.length; i++) {
    try {
      Module.FS.stat(SAVE_BASE_PATHS[i]);       // 目录存在则命中
      return SAVE_BASE_PATHS[i] + slot + '.sta';
    } catch(e) {}
  }
  return SAVE_BASE_PATHS[0] + slot + '.sta';    // 默认路径
}
```

### 2. 导出存档（`exportSave`）

从虚拟 FS 读取存档文件，通过浏览器下载到本地：

```javascript
function exportSave() {
  if (typeof Module === 'undefined' || !Module.FS) {
    setStatus('游戏尚未启动', true);
    return;
  }
  var slot = document.getElementById('save-slot').value;
  var path = findSavePath(slot);
  try {
    var data = Module.FS.readFile(path);
    var blob = new Blob([data], { type: 'application/octet-stream' });
    var a = document.createElement('a');
    a.href = URL.createObjectURL(blob);
    a.download = GAME_NAME + '_slot' + (parseInt(slot) + 1) + '.sta';
    a.click();
    setStatus('存档导出成功 ✓');
  } catch(e) {
    setStatus('未找到存档，请先在游戏内按 Shift+F7 保存', true);
    // 调试：打印所有候选目录内容
    SAVE_BASE_PATHS.forEach(function(p) {
      try { console.log('[Save] 目录内容:', p, Module.FS.readdir(p)); }
      catch(e2) { console.log('[Save] 目录不存在:', p); }
    });
  }
}
```

### 3. 导入存档（`importSave`）

将用户选择的本地 `.sta` 文件写入虚拟 FS，游戏内按 F7 即可读取：

```javascript
function importSave(input) {
  if (typeof Module === 'undefined' || !Module.FS) {
    setStatus('游戏尚未启动', true);
    return;
  }
  var file = input.files[0];
  if (!file) return;
  var slot = document.getElementById('save-slot').value;
  var reader = new FileReader();
  reader.onload = function(e) {
    var path = findSavePath(slot);
    // 确保目录存在
    var dir = path.substring(0, path.lastIndexOf('/'));
    try { Module.FS.mkdir(dir, 0o777); } catch(ex) {}
    Module.FS.writeFile(path, new Uint8Array(e.target.result));
    setStatus('导入成功，游戏内按 F7 读取槽位' + (parseInt(slot) + 1) + ' ✓');
  };
  reader.readAsArrayBuffer(file);
  input.value = ''; // 清空，允许重复选同一文件
}
```

### 4. 状态提示（`setStatus`）

```javascript
function setStatus(msg, isError) {
  var el = document.getElementById('save-status');
  el.textContent = msg;
  el.style.color = isError ? '#c00' : '#c80';
  clearTimeout(el._timer);
  el._timer = setTimeout(function() { el.textContent = ''; }, 3000);
}
```

---

## 使用流程

```
游戏运行中
    ↓
按 Shift+F7（在游戏内存档到槽位 N）
    ↓
选择对应槽位 → 点击「导出存档」
    ↓
浏览器下载 tengaij_slotN.sta 到本地
    ↓
（下次打开游戏后）
    ↓
选择槽位 → 点击「导入存档」→ 选择 .sta 文件
    ↓
游戏内按 F7 读档
```

---

## 注意事项

| 事项 | 说明 |
|------|------|
| 导出时机 | 必须先在游戏内按 `Shift+F7` 存档，再导出 |
| 导入时机 | 导入后需在游戏内按 `F7` 读档才生效，不会自动加载 |
| 文件格式 | `.sta` 是 MAME 专有存档格式，仅限同版本同驱动使用 |
| 槽位对应 | UI 槽位 1/2/3 对应 MAME 内部槽位 0/1/2（文件名 `0.sta`/`1.sta`/`2.sta`） |
| 存档路径 | 通过候选路径列表自动探测，兼容不同 MAME 编译版本 |

---

## 移植到其他游戏

只需修改一处配置：

```javascript
var GAME_NAME = 'shadfrceu';  // 改为目标游戏的驱动名
```

存档路径、导出文件名均会自动跟随 `GAME_NAME` 变化。
