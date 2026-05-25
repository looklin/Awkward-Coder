---
title: bwip-js 实战 — 用 JavaScript 搞定一切条形码和二维码
slug: bwip-js-barcode-generation-practice
description: >-
  从安装到实战，一文掌握 bwip-js 在 Node.js 和浏览器端的条码生成能力：商品条码、物流标签、二维码、批量生成等场景。
tags:
  - technical
added: "May 25 2026"
---

## 引言

做项目时遇到条码生成需求，你的第一反应是什么？

- 「找个在线生成器截图」→ 不靠谱
- 「用 Java/Python 库」→ 太重
- 「自己画 Canvas」→ 头大

其实，一个 **npm install** 就能搞定一切。这就是今天的主角 —— **bwip-js**。

> 项目地址：[github.com/metafloor/bwip-js](https://github.com/metafloor/bwip-js)
> 支持 100+ 种条码格式，零依赖，Node.js 和浏览器都能跑。

---

## 一、bwip-js 是什么？

**bwip-js**（Browser/Window Image Processor for JavaScript）是一个纯 JavaScript 条码生成库。它不依赖任何外部库或字体文件，完全用 JS 从零绘制条码像素。

**核心能力：**

| 能力 | 说明 |
|------|------|
| **100+ 条码格式** | QR Code、Code 128、EAN-13、DataMatrix、PDF417 等 |
| **Node.js + 浏览器** | 同一套 API，两边都能用 |
| **零依赖** | 不需要安装任何额外包 |
| **多种输出** | PNG Buffer / Canvas / SVG / Base64 |
| **轻量** | 核心库约 300KB |

**能生成的常见条码类型：**

```
商品零售：    EAN-13、EAN-8、UPC-A、UPC-E
物流运输：    Code 128、Code 39、ITF-14
二维码：      QR Code、DataMatrix、PDF417
医疗工业：    GS1-128、HIBC、Pharmacode
其他：        Codabar、ISBN、ISSN、Telepen...
```

---

## 二、安装

```bash
npm install bwip-js
```

是的，就这么简单。零依赖，一个包搞定。

---

## 三、Node.js 实战

### 3.1 生成第一个二维码

```javascript
const bwipjs = require('bwip-js');

bwipjs.toBuffer({
    bcid:       'qrcode',       // 条码类型
    text:       'Hello, World!', // 条码内容
    scale:      3,               // 缩放倍数
    includetext: true,           // 是否显示文字
    textxalign: 'center',        // 文字居中
}, (err, png) => {
    if (err) {
        console.error('生成失败:', err);
        return;
    }
    // png 是一个 Buffer，可以直接写入文件
    require('fs').writeFileSync('qrcode.png', png);
    console.log('二维码已生成！');
});
```

运行后会在当前目录生成 `qrcode.png`，手机扫码就能看到 "Hello, World!"。

### 3.2 生成商品条码（EAN-13）

```javascript
const bwipjs = require('bwip-js');
const fs = require('fs');

// EAN-13 是 13 位数字，最后一位是校验位（可以自动计算）
bwipjs.toBuffer({
    bcid:       'ean13',
    text:       '690123456789',   // 12 位，自动补校验位
    scale:      3,
    includetext: true,
    textxalign: 'center',
    height:     10,                // 条码高度（mm）
    width:      2,                 // 窄条宽度
}, (err, png) => {
    if (err) throw err;
    fs.writeFileSync('ean13.png', png);
    console.log('商品条码已生成！');
});
```

### 3.3 生成物流标签（Code 128）

```javascript
bwipjs.toBuffer({
    bcid:       'code128',
    text:       'SF1234567890CN',  // 快递单号
    scale:      3,
    includetext: true,
    textxalign: 'center',
    backgroundcolor: 'FFFFFF',     // 背景色
    barcolor:     '000000',        // 条码颜色
}, (err, png) => {
    if (err) throw err;
    require('fs').writeFileSync('tracking.png', png);
});
```

### 3.4 批量生成条码

实际项目中经常需要批量生成，比如给一批商品打标签：

```javascript
const bwipjs = require('fs').promises ? require('bwip-js') : require('bwip-js');
const fs = require('fs').promises;
const path = require('path');

// 批量生成商品条码
const products = [
    { name: '商品A', ean: '690111111111' },
    { name: '商品B', ean: '690222222222' },
    { name: '商品C', ean: '690333333333' },
    { name: '商品D', ean: '690444444444' },
    { name: '商品E', ean: '690555555555' },
];

async function generateBarcodes(products, outputDir) {
    // 确保输出目录存在
    await fs.mkdir(outputDir, { recursive: true });

    const results = await Promise.allSettled(
        products.map(async (product) => {
            const png = await bwipjs.toBuffer({
                bcid:       'ean13',
                text:       product.ean,
                scale:      3,
                includetext: true,
                textxalign: 'center',
            });

            const filename = path.join(outputDir, `${product.name}.png`);
            await fs.writeFile(filename, png);
            return { name: product.name, status: 'success' };
        })
    );

    results.forEach((result, index) => {
        if (result.status === 'fulfilled') {
            console.log(`✅ ${products[index].name} 生成成功`);
        } else {
            console.log(`❌ ${products[index].name} 生成失败: ${result.reason.message}`);
        }
    });
}

generateBarcodes(products, './barcodes');
```

### 3.5 Express 接口：实时生成条码

更实用的场景是提供一个 API 接口，前端请求时实时生成：

```javascript
const express = require('express');
const bwipjs = require('bwip-js');
const app = express();

// 实时生成条码接口
app.get('/api/barcode', (req, res) => {
    const { type = 'qrcode', text, scale = 3 } = req.query;

    if (!text) {
        return res.status(400).json({ error: '缺少 text 参数' });
    }

    bwipjs.toBuffer({
        bcid:       type,
        text:       text,
        scale:      Number(scale),
        includetext: true,
        textxalign: 'center',
    }, (err, png) => {
        if (err) {
            return res.status(400).json({ error: err.message });
        }

        // 直接返回图片
        res.setHeader('Content-Type', 'image/png');
        res.send(png);
    });
});

// 使用示例：
// 二维码：  http://localhost:3000/api/barcode?type=qrcode&text=Hello
// 商品条码：http://localhost:3000/api/barcode?type=ean13&text=690123456789
// 物流码：  http://localhost:3000/api/barcode?type=code128&text=SF1234567890CN

app.listen(3000, () => console.log('服务启动：http://localhost:3000'));
```

前端直接用 `<img>` 标签调用：

```html
<!-- 二维码 -->
<img src="/api/barcode?type=qrcode&text=Hello+World">

<!-- 商品条码 -->
<img src="/api/barcode?type=ean13&text=690123456789">
```

---

## 四、浏览器端实战

### 4.1 Canvas 方式

bwip-js 可以直接在浏览器的 `<canvas>` 上绘制条码：

```html
<!DOCTYPE html>
<html>
<head>
    <title>条码生成</title>
    <script src="https://unpkg.com/bwip-js/dist/bwip-js-min.js"></script>
    <style>
        body { font-family: sans-serif; padding: 20px; }
        .container { max-width: 600px; margin: 0 auto; }
        canvas { border: 1px solid #ddd; margin: 10px 0; }
        input, select, button { padding: 8px; margin: 5px 0; }
    </style>
</head>
<body>
    <div class="container">
        <h2>条码生成器</h2>

        <div>
            <label>类型：</label>
            <select id="barcodeType">
                <option value="qrcode">二维码 (QR Code)</option>
                <option value="ean13">商品条码 (EAN-13)</option>
                <option value="code128">物流条码 (Code 128)</option>
                <option value="datamatrix">DataMatrix</option>
            </select>
        </div>

        <div>
            <label>内容：</label>
            <input type="text" id="barcodeText" value="Hello, World!" 
                   style="width: 300px;">
        </div>

        <div>
            <label>缩放：</label>
            <input type="number" id="barcodeScale" value="3" min="1" max="10">
        </div>

        <button onclick="generate()">生成条码</button>

        <canvas id="barcodeCanvas"></canvas>

        <button onclick="download()">下载 PNG</button>
    </div>

    <script>
        function generate() {
            const type = document.getElementById('barcodeType').value;
            const text = document.getElementById('barcodeText').value;
            const scale = parseInt(document.getElementById('barcodeScale').value);
            const canvas = document.getElementById('barcodeCanvas');

            if (!text) {
                alert('请输入条码内容');
                return;
            }

            try {
                bwipjs.toCanvas(canvas, {
                    bcid:        type,
                    text:        text,
                    scale:       scale,
                    includetext: true,
                    textxalign:  'center',
                });
            } catch (e) {
                alert('生成失败：' + e.message);
            }
        }

        function download() {
            const canvas = document.getElementById('barcodeCanvas');
            const link = document.createElement('a');
            link.download = 'barcode.png';
            link.href = canvas.toDataURL('image/png');
            link.click();
        }

        // 页面加载后自动生成一次
        generate();
    </script>
</body>
</html>
```

### 4.2 React 组件封装

```jsx
import { useEffect, useRef } from 'react';
import bwipjs from 'bwip-js';

export default function Barcode({
    type = 'qrcode',
    text = '',
    scale = 3,
    width = 200,
    height = 200,
}) {
    const canvasRef = useRef(null);

    useEffect(() => {
        if (!text || !canvasRef.current) return;

        try {
            bwipjs.toCanvas(canvasRef.current, {
                bcid:        type,
                text:        text,
                scale:       scale,
                includetext: true,
                textxalign:  'center',
            });
        } catch (err) {
            console.error('条码生成失败:', err);
        }
    }, [type, text, scale]);

    return (
        <canvas
            ref={canvasRef}
            width={width}
            height={height}
            style={{ border: '1px solid #eee' }}
        />
    );
}
```

使用方式：

```jsx
import Barcode from './Barcode';

// 二维码
<Barcode type="qrcode" text="https://example.com" scale={4} />

// 商品条码
<Barcode type="ean13" text="690123456789" scale={3} />

// 物流条码
<Barcode type="code128" text="SF1234567890CN" scale={2} />
```

---

## 五、实战场景汇总

### 5.1 电商：商品标签打印

```javascript
// 生成带商品信息的标签
function generateProductLabel(product) {
    return bwipjs.toBuffer({
        bcid:        'ean13',
        text:        product.ean,
        scale:       3,
        includetext: true,
        textxalign:  'center',
        textfont:    'Helvetica',
        textsize:    10,
        textgaps:    1,
        backgroundcolor: 'FFFFFF',
    });
}
```

### 5.2 仓储：入库标签（含 DataMatrix）

```javascript
// DataMatrix 适合小空间存储大量信息
bwipjs.toBuffer({
    bcid:   'datamatrix',
    text:   `SKU:ABC-12345|LOC:A-01-02|QTY:100|DATE:2026-05-25`,
    scale:  4,
    backgroundcolor: 'FFFFFF',
}, (err, png) => {
    // 打印或保存到文件
});
```

### 5.3 票务：PDF417 登机牌/门票

```javascript
// PDF417 常用于登机牌、演唱会门票
bwipjs.toBuffer({
    bcid:   'pdf417',
    text:   'TICKET:VIP-20260525-001|NAME:张三|SEAT:A-12',
    scale:  3,
    columns: 5,   // 列数
    rowmult: 3,   // 行高倍数
}, (err, png) => {
    require('fs').writeFileSync('ticket.png', png);
});
```

### 5.4 微信跳转码

```javascript
// 生成微信 Open URL 跳转二维码
bwipjs.toBuffer({
    bcid:   'qrcode',
    text:   'weixin://dl/business/?ticket=xxxxx',
    scale:  5,
    eclevel: 'H',   // 高容错率（支持部分遮挡仍能识别）
}, (err, png) => {
    require('fs').writeFileSync('wechat-qrcode.png', png);
});
```

---

## 六、常用参数速查

| 参数 | 说明 | 示例值 |
|------|------|--------|
| `bcid` | 条码类型 | `'qrcode'`, `'ean13'`, `'code128'` |
| `text` | 条码内容 | `'Hello'`, `'690123456789'` |
| `scale` | 缩放倍数 | `3` |
| `width` | 窄条宽度 | `2` |
| `height` | 条码高度 | `10` |
| `includetext` | 是否显示文字 | `true` |
| `textxalign` | 文字对齐 | `'center'`, `'left'`, `'right'` |
| `textfont` | 字体 | `'Helvetica'` |
| `textsize` | 字号 | `10` |
| `barcolor` | 条码颜色 | `'000000'` (黑) |
| `backgroundcolor` | 背景色 | `'FFFFFF'` (白) |
| `eclevel` | QR 容错率 | `'L'`, `'M'`, `'Q'`, `'H'` |
| `padding` | 内边距 | `10` |
| `backgroundimage` | 背景图 | `Buffer / URL` |

---

## 七、常见问题

### Q1：生成的条码扫不出来？

- 检查条码内容是否符合格式要求（EAN-13 必须是数字）
- `scale` 太小会导致条码太密，建议至少 2
- 二维码内容太长时，调大 `scale` 和 `eclevel`

### Q2：中文内容怎么办？

EAN-13、Code 128 等一维码不支持中文。但 **QR Code 和 DataMatrix 支持 UTF-8**：

```javascript
bwipjs.toBuffer({
    bcid:   'qrcode',
    text:   '你好，世界！',
    scale:  4,
});
```

### Q3：Node.js 和浏览器端 API 有什么区别？

核心参数完全一样，只是输出方式不同：

- Node.js：`toBuffer()` → 返回 PNG Buffer
- 浏览器：`toCanvas()` → 绘制到 `<canvas>`

---

## 总结

bwip-js 是 JavaScript 生态里条码生成的瑞士军刀：

- **一个包**覆盖 100+ 种条码格式
- **两端通用**，Node.js 和浏览器同一套 API
- **零依赖**，npm install 就能用
- **轻量高效**，生成一个条码只要几毫秒

下次遇到条码需求，别再造轮子了，一个 `bwipjs.toBuffer()` 搞定。

---

> 参考资料：
> - [bwip-js GitHub](https://github.com/metafloor/bwip-js)
> - [在线演示](http://metafloor.github.io/bwip-js/demo/demo.html)
