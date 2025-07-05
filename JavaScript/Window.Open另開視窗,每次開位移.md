您的需求是每次開啟新的視窗時，都希望它的位置能夠略微偏移，這樣可以讓多個視窗並排顯示，而不是完全重疊在一起。

以下是經過整理與補充的 JavaScript 代碼，會讓每次開啟視窗時，視窗的位置略微偏移，從而避免覆蓋現有的視窗。

### 完整範例程式碼：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Multiple Popup Windows</title>
</head>
<body>
    <button id="aaaa">Open Window</button>

    <script>
        var popupWindowCount = 0;  // 用來追蹤已開啟視窗的數量

        document.getElementById('aaaa').addEventListener('click', function () {
            var windowWidth = 500;  // 視窗的寬度
            var windowHeight = 500; // 視窗的高度

            // 每個視窗的偏移量，這裡會根據已開啟的視窗數量來設定
            var left = 100 + popupWindowCount * 20; // 每開一個視窗，向右偏移 20px
            var top = 100 + popupWindowCount * 20;  // 每開一個視窗，向下偏移 20px

            // 打開新的視窗
            var popupWindow = window.open('/home/index', '_blank', `left=${left},top=${top},width=${windowWidth},height=${windowHeight}`);

            // 每次開啟視窗後，視窗數量加 1
            popupWindowCount++;
        });
    </script>
</body>
</html>
```

### 程式碼說明：

1. **視窗大小設定**：

   * `windowWidth = 500` 和 `windowHeight = 500` 設定了每個視窗的大小為 500x500。

2. **位置偏移設計**：

   * `var left = 100 + popupWindowCount * 20;` 設定了每個新視窗的 `left` 偏移，隨著視窗數量的增加，`left` 會增加 20 像素，這樣視窗會有水平的間距。
   * `var top = 100 + popupWindowCount * 20;` 同理，`top` 偏移會根據視窗數量增加，讓每個新視窗略微向下偏移，這樣可以防止所有視窗重疊。

3. **視窗開啟**：

   * 使用 `window.open()` 開啟新的視窗，並根據 `left` 和 `top` 計算位置，這樣每次開啟的新視窗都有一點點的偏移。

4. **視窗數量**：

   * `popupWindowCount` 用來追蹤已開啟的視窗數量，確保每次開啟新視窗時，位置都會根據已開啟的視窗數量進行偏移。

### 預覽效果：

每次點擊「Open Window」按鈕，都會開啟一個新的 500x500 的視窗，且每個視窗的顯示位置會略微向右下角偏移，這樣可以清晰地看到多個視窗。

您可以直接將上述程式碼放入 HTML 文件中，並在瀏覽器中測試，點擊按鈕開啟多個視窗，您會發現視窗位置會逐漸偏移，防止視窗完全重疊。
