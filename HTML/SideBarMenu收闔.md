
### 完整的 HTML 與 CSS 程式碼：

#### HTML

```html
<div id="wrapper">
  <div class="child child1">
    111
  </div>
  <div class="child child2">
    222
  </div>
</div>
```

#### CSS

```css
#wrapper {
  margin-left: 200px;      /* 使父容器左側有 200px 的間距 */
  width: 200px;            /* 設定父容器的寬度為 200px */
  height: 200px;           /* 設定父容器的高度為 200px */
  background: lightblue;   /* 設定父容器的背景顏色 */
  display: flex;           /* 使用 Flexbox 來排列子元素 */
}

.child {
  height: 100%;            /* 子元素高度為父容器的 100% */
  display: inline-block;   /* 子元素的顯示方式為 inline-block */
}

.child1 {
  margin-left: -10px;      /* 左側縮排 10px，可以調整該值來縮放或移動子元素 */
  width: 50px;             /* 子元素1的寬度為 50px */
  background-color: lightgreen; /* 設定子元素1的背景顏色 */
}

.child2 {
  width: 100%;             /* 子元素2的寬度為父容器的 100% */
  background-color: lightpink; /* 設定子元素2的背景顏色 */
}
```

### 解釋與補充

1. **父容器 `#wrapper`**：

   * `margin-left: 200px;`：為父容器 `#wrapper` 添加了左邊的 200px 間距，將其從頁面左邊邊緣偏移。
   * `width: 200px;` 和 `height: 200px;`：將父容器的大小設為 200x200 像素。
   * `background: lightblue;`：設置父容器的背景顏色為淺藍色。
   * `display: flex;`：使用 Flexbox 來排列其子元素，這樣子元素就會被水平排列（默認為水平方向）。

2. **子元素 `.child`**：

   * `height: 100%;`：每個子元素的高度都會與父容器相同，即 100%。
   * `display: inline-block;`：這使得子元素呈現為內聯區塊元素，這樣它們會按順序排列並且可以設置寬度和高度。

3. **子元素 `.child1`**：

   * `margin-left: -10px;`：這個負值的左邊距使得 `.child1` 向左偏移 10px。這個屬性可以用來縮放或調整子元素之間的間距。根據需求調整此值會影響它與 `.child2` 之間的重疊。
   * `width: 50px;`：將 `.child1` 的寬度設為 50px。
   * `background-color: lightgreen;`：將 `.child1` 的背景顏色設為淺綠色。

4. **子元素 `.child2`**：

   * `width: 100%;`：這會讓 `.child2` 的寬度占滿剩餘的空間（即父容器的 100%），由於 `.child1` 使用了 50px 寬度，`child2` 則會填滿剩下的空間。
   * `background-color: lightpink;`：將 `.child2` 的背景顏色設為淺粉色。

### 效果說明

* 父容器 (`#wrapper`) 具有固定的 200x200px 大小，並使用 Flexbox 排列其子元素。
* `.child1` 的寬度為 50px，並且透過負的 `margin-left` 進行偏移，這樣使得它的左側稍微向左移動。
* `.child2` 的寬度佔滿剩餘的空間，即 150px（200px - 50px）並填充剩餘的區域，從而確保 `.child1` 和 `.child2` 在水平方向上並排顯示。

### 建議與改進

1. **布局調整**：

   * 若您希望讓 `.child1` 和 `.child2` 之間有一定的間距，可以適當調整 `margin-left` 和 `width` 屬性，來達到所需的布局效果。
   * 使用 `justify-content` 和 `align-items` 屬性可以進一步控制元素的對齊方式。

2. **靈活配置**：

   * 若希望支持響應式設計，您可以使用 `flex-wrap: wrap;` 來使元素在空間不足時自動換行，從而提高適應不同屏幕尺寸的能力。

### 修改後的示例：

```css
#wrapper {
  margin-left: 200px;
  width: 200px;
  height: 200px;
  background: lightblue;
  display: flex;
  justify-content: space-between; /* 在水平方向上讓元素有間隔 */
}

.child {
  height: 100%;
  display: inline-block;
}

.child1 {
  margin-left: -10px;
  width: 50px;
  background-color: lightgreen;
}

.child2 {
  width: 140px;  /* 增加 child2 的寬度，避免讓其變形 */
  background-color: lightpink;
}
```

這樣的設置會使得 `#wrapper` 內的子元素更加靈活並且易於調整，無論是對齊還是間距的管理都變得更簡單。
