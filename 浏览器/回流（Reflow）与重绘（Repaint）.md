### 回流（Reflow）与重绘（Repaint）详解

---

#### **1. 概念**
**回流（Reflow）**：  
当渲染树（Render Tree）中的元素的尺寸、位置、布局等几何属性发生变化时，浏览器需要重新计算元素的几何信息，并更新页面布局的过程。**回流一定会触发后续的重绘**。

**重绘（Repaint）**：  
当元素的样式（如颜色、背景、边框等）发生变化，但不影响布局时，浏览器只需重新绘制受影响的部分到屏幕上，无需重新计算布局。**重绘不一定会触发回流**。

---

#### **2. 触发回流或重绘的操作**

##### **触发回流的常见操作**：
1. **几何属性变化**：
   - 修改元素的宽度、高度、边距（`width`, `height`, `margin`, `padding`）。
   - 改变元素的定位方式（`position: static → fixed`）。
   - 调整布局属性（`display: none → block`, `flex` 容器变化等）。

2. **内容变化**：
   - 文本内容增减（如输入框输入文字）。
   - 图片大小改变（加载完成或动态修改 `src`）。

3. **视口或窗口变化**：
   - 浏览器窗口大小调整（`resize` 事件）。
   - 设备方向变化（横竖屏切换）。

4. **读取布局属性**：
   - 访问某些属性（如 `offsetTop`、`scrollHeight`、`clientWidth`）时，可能会使浏览器执行回流以获取最新值（比如在同步代码中先写后读，见《3. 回流与重绘的时机》）。

##### **触发重绘的常见操作**：
1. **样式变化不影响布局**：
   - 修改颜色（`color`, `background-color`）。
   - 调整透明度（`opacity`，注意：`opacity=0` 不会触发回流）。
   - 改变边框样式（`border-style`, `outline`）。
   - 阴影变化（`box-shadow`）。

---

#### **3. 回流与重绘的时机**
浏览器的渲染流程是 **异步** 的，但某些操作会强制同步触发回流：

##### **浏览器优化机制**：
- 浏览器会将多次样式修改操作放入队列，批量处理以减少回流次数。但以下操作会强制刷新队列，立即触发回流：
  - 访问以下布局属性（触发 **同步回流**）：
    ```javascript
    offsetTop, offsetLeft, offsetWidth, offsetHeight
    scrollTop, scrollLeft, scrollWidth, scrollHeight
    clientTop, clientLeft, clientWidth, clientHeight
    getComputedStyle(), getBoundingClientRect()
    ```
  - 修改样式后立即读取上述属性（如先写后读）。

##### **典型场景示例**：
```javascript
// 触发两次回流（未优化）
const el = document.getElementById('box');
el.style.width = '100px'; // 修改样式（入队）
const width = el.offsetWidth; // 读取属性（强制刷新队列，触发回流）
el.style.height = '200px'; // 再次修改（入队）
const height = el.offsetHeight; // 再次触发回流

// 优化后（合并读写操作）
el.style.width = '100px';
el.style.height = '200px';
const width = el.offsetWidth; // 只触发一次回流
```

---

#### **4. 性能优化建议**
1. **减少频繁操作样式**：
   - 使用 `class` 批量修改样式，而非逐行修改 `style`。
   - 对 DOM 进行离线操作（如 `display: none` 后修改，再显示）。

2. **避免强制同步布局**：
   - 将读取布局属性的操作放在所有样式修改之后。

3. **使用 CSS 优化**：
   - 使用 `transform` 和 `opacity` 实现动画（跳过布局计算，直接进入合成阶段）。
   - 使用 `will-change: transform` 将元素提升为独立图层。

4. **优化 JavaScript**：
   - 使用 `DocumentFragment` 批量插入 DOM 节点。
   - 避免在循环中频繁操作样式。

---

#### **5. 浏览器渲染流程回顾**
1. **解析 HTML/CSS** → 生成 DOM 树和 CSSOM 树。
2. **合并为渲染树**（Render Tree）。
3. **布局（Layout）**：计算元素位置和大小（触发回流）。
4. **绘制（Paint）**：填充像素（触发重绘）。
5. **合成（Composite）**：将各图层合并显示（如 `transform` 直接进入此阶段）。

---

通过理解回流与重绘的机制，开发者可以显著提升页面性能，尤其是在高频操作（如动画、滚动）中减少不必要的计算。