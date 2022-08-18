---
title: Web前端实现复制到剪切板/剪贴板/粘贴板
date: 2015-08-31 23:41:14
tags: [复制到剪切板]
---
## 目录

*   [document.execCommand()](#documentexeccommand)

*   [Clipboard API](#clipboard-api)

# Web前端实现复制到剪切板/剪贴板/粘贴板

## document.execCommand()

1.  使用 `document.execCommand('copy')` 将文本**复制到剪贴板**中。

2.  使用 `document.execCommand('cut')` 剪切文本并将其添加**到剪贴板**中。

3.  使用 `document.execCommand('paste')` 粘贴已经出现在**剪贴板**上的内容。

> 但 `document.execCommand()` 接口已被标记为 `废弃` ，未来可能就直接不支持了。

```javascript
var textarea = document.createElement('textarea');
document.body.appendChild(textarea);
// 隐藏此输入框
textarea.style.position = 'fixed';
textarea.style.clip = 'rect(0 0 0 0)';
textarea.style.top = '10px';
// 赋值
textarea.value = text;
// 选中
textarea.select();
// 复制
document.execCommand('copy', true);
// 移除输入框
document.body.removeChild(textarea);
```

## Clipboard API

[https://developer.mozilla.org/zh-CN/docs/Web/API/Clipboard/writeText](https://developer.mozilla.org/zh-CN/docs/Web/API/Clipboard/writeText "https://developer.mozilla.org/zh-CN/docs/Web/API/Clipboard/writeText")

[Clipboard](https://developer.mozilla.org/zh-CN/docs/Web/API/Clipboard "Clipboard")接口的  方法可以写入特定字符串到操作系统的剪切板。

> 问题：该接口只支持在安全环境（HTTPS）下运行。如果需要兼容非安全环境，可以结合 `document.execCommand()` 命令

```javascript
// return a promise
function copyToClipboard(textToCopy) {
    // navigator.clipboard需要https才能正常运行
    if (navigator.clipboard && window.isSecureContext) {
        return navigator.clipboard.writeText(textToCopy);
    } else {
        // 如果不是安全环境（非https），使用 document.execCommand('copy')
        let textArea = document.createElement("textarea");
        textArea.value = textToCopy;
        // make the textarea out of viewport
        textArea.style.position = "fixed";
        textArea.style.left = "-999999px";
        textArea.style.top = "-999999px";
        document.body.appendChild(textArea);
        textArea.focus();
        textArea.select();
        return new Promise((res, rej) => {
            // document.execCommand('copy')这个接口后面可能会被丢弃
            document.execCommand('copy') ? res() : rej();
            textArea.remove();
        });
    }
}

// 如何使用
copyToClipboard("I'm going to the clipboard !")
    .then(() => console.log('text copied !'))
    .catch(() => console.log('error'));
```
