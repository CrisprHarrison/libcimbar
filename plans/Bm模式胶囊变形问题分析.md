# Bm 模式胶囊变形问题分析

**日期**: 2024-12-12  
**问题**: 在没有文件的状态下，点击 Bm 模式会导致胶囊变大或变形，而 B 和 4C 模式不会

---

## 🔍 问题表现

### 用户反馈
- ✅ **B 模式**：点击后胶囊保持不变
- ✅ **4C 模式**：点击后胶囊保持不变  
- ❌ **Bm 模式**：点击后胶囊会变大或变形

---

## 📊 代码分析

### 1. setMode() 函数流程

查看 [`main_v2.js:243-273`](web/pack/v3/main_v2.js:243-273)：

```javascript
setMode: function (mode_str) {
  let modeVal = 68;  // 默认 B 模式
  if (mode_str == "4C") {
    modeVal = 4;
  }
  else if (mode_str == "Bm") {
    modeVal = 67;
  }
  
  // 关键步骤 1: 配置 WebAssembly 模块
  Module._cimbare_configure(modeVal, -1);
  
  // 关键步骤 2: 获取新的宽高比
  _idealRatio = Module._cimbare_get_aspect_ratio();
  
  // 关键步骤 3: 调用 resize()
  Main.resize();
  
  // 更新导航栏样式...
}
```

### 2. resize() 函数逻辑

查看 [`main_v2.js:85-93`](web/pack/v3/main_v2.js:85-93)：

```javascript
resize: function () {
  // reset zoom
  var canvas = document.getElementById('canvas');
  var width = window.innerWidth - 200;
  var height = window.innerHeight - 50;
  Main.scaleCanvas(canvas, width, height);
  Main.alignInvisibleClick(canvas);
}
```

### 3. scaleCanvas() 函数逻辑

查看 [`main_v2.js:112-138`](web/pack/v3/main_v2.js:112-138)：

```javascript
scaleCanvas: function (canvas, width, height) {
  // 使用当前配置的宽高比
  var needRotate = _idealRatio > 1 && height > width;
  Module._cimbare_rotate_window(needRotate);

  var ourRatio = needRotate ? height / width : width / height;

  var xdim = needRotate ? height : width;
  var ydim = needRotate ? width : height;
  
  // 根据宽高比调整尺寸
  if (ourRatio > _idealRatio) {
    xdim = Math.floor(xdim * _idealRatio / ourRatio);
  }
  else if (ourRatio < _idealRatio) {
    ydim = Math.floor(ydim * ourRatio / _idealRatio);
  }

  console.log(xdim + "x" + ydim);
  
  // 设置 Canvas 样式
  if (needRotate) {
    canvas.style.width = ydim + "px";
    canvas.style.height = xdim + "px";
  }
  else {
    canvas.style.width = xdim + "px";
    canvas.style.height = ydim + "px";
  }
}
```

---

## 🎯 根本原因

### 问题核心

**`scaleCanvas()` 函数会修改 Canvas 的 `style.width` 和 `style.height`**，即使没有文件加载！

### 为什么只有 Bm 模式有问题？

关键在于 **`_idealRatio`（理想宽高比）** 的值：

1. **B 模式** (modeVal = 68)
   - `_idealRatio` = 某个值 A
   - 调用 `scaleCanvas()` 后，Canvas 尺寸 = X

2. **4C 模式** (modeVal = 4)
   - `_idealRatio` = 某个值 B
   - 调用 `scaleCanvas()` 后，Canvas 尺寸 = Y

3. **Bm 模式** (modeVal = 67)
   - `_idealRatio` = 某个值 C（**与 A 和 B 不同**）
   - 调用 `scaleCanvas()` 后，Canvas 尺寸 = Z（**明显不同**）

### 视觉影响

由于 Canvas 是 dragdrop 容器的子元素：

```html
<div id="dragdrop" class="dragdrop">
  <canvas id="canvas" class="canvas"></canvas>
</div>
```

当 Canvas 的尺寸被修改时，即使 dragdrop 本身的 CSS 没变，**视觉上也会感觉容器发生了变化**。

---

## 💡 解决方案

### 方案 1: 在 resize() 中添加文件检查（已被用户否决）

```javascript
resize: function () {
  var dragdrop = document.getElementById('dragdrop');
  if (!dragdrop.classList.contains('has-file')) {
    return;  // ❌ 用户说这个方案有问题
  }
  // ...
}
```

**问题**: 用户反馈这个实现方式有问题，所以删掉了。

### 方案 2: 在 setMode() 中添加文件检查 ✅

```javascript
setMode: function (mode_str) {
  let modeVal = 68;
  if (mode_str == "4C") {
    modeVal = 4;
  }
  else if (mode_str == "Bm") {
    modeVal = 67;
  }
  
  Module._cimbare_configure(modeVal, -1);
  _idealRatio = Module._cimbare_get_aspect_ratio();
  
  // ✅ 只有加载文件后才调用 resize
  var dragdrop = document.getElementById('dragdrop');
  if (dragdrop.classList.contains('has-file')) {
    Main.resize();
  }
  
  // 更新导航栏样式...
}
```

**优点**:
- 精确控制：只在需要时调用 resize
- 不影响其他地方调用 resize() 的逻辑
- 保持 B、Bm、4C 三个模式行为一致

### 方案 3: 初始化时隐藏 Canvas ✅

```css
#dragdrop:not(.has-file) #canvas {
  display: none;
  /* 或者 */
  visibility: hidden;
  width: 0;
  height: 0;
}
```

**优点**:
- 纯 CSS 解决方案
- 无文件时 Canvas 完全不可见
- 不影响 JavaScript 逻辑

### 方案 4: 组合方案（推荐）✅

结合方案 2 和方案 3：

**JavaScript 改动**:
```javascript
setMode: function (mode_str) {
  // ... 配置代码 ...
  
  // 只有加载文件后才调用 resize
  var dragdrop = document.getElementById('dragdrop');
  if (dragdrop.classList.contains('has-file')) {
    Main.resize();
  }
  
  // ... 导航栏样式更新 ...
}
```

**CSS 改动**:
```css
/* 无文件时隐藏 Canvas */
#dragdrop:not(.has-file) #canvas {
  display: none;
}
```

---

## 🔬 深入分析：为什么 Bm 的宽高比不同？

### 可能的原因

1. **编码模式差异**
   - B 模式：标准编码
   - Bm 模式：可能是 "B mode with modifications"
   - 4C 模式：4色编码

2. **数据密度不同**
   - 不同模式可能使用不同的 cell 布局
   - 导致最优的显示宽高比不同

3. **WebAssembly 配置**
   - `Module._cimbare_get_aspect_ratio()` 返回值取决于 `modeVal`
   - Bm (67) 的返回值可能与 B (68) 和 4C (4) 显著不同

---

## 📋 实施建议

### 推荐方案：方案 4（组合方案）

**理由**:
1. **防御性编程**: JavaScript 检查 + CSS 隐藏双重保护
2. **用户体验**: 无文件时 Canvas 完全不可见，避免任何视觉干扰
3. **代码清晰**: 逻辑明确，易于维护
4. **性能优化**: 无文件时不执行不必要的 Canvas 计算

### 实施步骤

1. ✅ 修改 [`main_v2.js`](web/pack/v3/main_v2.js) 的 [`setMode()`](web/pack/v3/main_v2.js:243) 函数
2. ✅ 修改 [`index_v2.html`](web/pack/v3/index_v2.html) 的 CSS 样式
3. ✅ 测试三个模式在无文件状态下的行为
4. ✅ 测试加载文件后的模式切换功能
5. ✅ 更新开发日志

---

## 🎓 经验总结

### 教训

1. **状态管理很重要**: 需要明确区分"有文件"和"无文件"两种状态
2. **副作用要控制**: `resize()` 这样的函数应该只在必要时调用
3. **CSS 可以帮忙**: 很多视觉问题可以用 CSS 解决，不一定需要 JavaScript

### 最佳实践

1. **状态驱动**: 使用 CSS 类（如 `.has-file`）管理状态
2. **条件执行**: 在调用有副作用的函数前检查状态
3. **防御性 CSS**: 使用 `:not()` 选择器隐藏不需要的元素

---

**文档创建时间**: 2024-12-12  
**状态**: 待实施