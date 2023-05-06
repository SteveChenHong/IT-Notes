我要用js的window.open做另開視窗，大概500 * 500 ，我每另開一個視窗，視窗位置都要位移一點點讓我知道有多個視窗，而不是每開一個就整個覆蓋住上一個，會以為只有一個視窗在畫面上


以下是一個範例程式碼：

```
var popupWindow = {};
        document.getElementById('aaaa')
        .addEventListener('click', function () {
            var left = (popupWindow.screenX || 250) + 10
            var top = (popupWindow.screenY || 250) + 10
            popupWindow = window.open('/home/index', '_blank', `left=${left},top=${top},width=320,height=320`);
        });
```

