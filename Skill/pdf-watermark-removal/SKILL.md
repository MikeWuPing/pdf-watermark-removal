---
name: pdf-watermark-removal
description: Use when PDF contains watermarks that need removal, especially watermarks with custom fonts like AAAAAB+Helvetica-Bold, or when asked to clean/remove watermarks from PDF documents
---

# PDF 水印移除

## 概述

通过分析 PDF 内部结构，识别并移除水印 XObject，同时保持正常内容完整。核心原理：水印通常是 Form XObject，通过 Do 命令绘制到页面，通过检测水印特征字体找到对应的 XObject 并移除其绘制命令。

## 何时使用

- PDF 文档包含需要移除的水印
- 水印使用特定字体（如 AAAAAB+Helvetica-Bold）
- 需要批量处理 PDF 水印移除

**不适用于：** 扫描件水印（图像层）、加密 PDF

## 核心原理

```
Page → Resources → XObject → 水印XObject → 包含水印字体
                 ↘ Contents → Do命令 → 绘制水印XObject
```

水印通常存在于 Form XObject 中，页面内容流通过 `q ... cm /Name Do Q` 命令块绘制。

## 快速参考

| 步骤 | 操作 |
|------|------|
| 1. 分析 | 检查 XObject 是否包含水印字体 |
| 2. 定位 | 记录水印 XObject 的名称 |
| 3. 移除 | 从内容流中删除 Do 命令块 |

## 实现模式

### 检测水印 XObject

```python
def check_xobject_has_watermark_font(obj, depth=0, max_depth=10):
    """递归检查 XObject 是否包含水印字体"""
    if depth > max_depth:
        return False

    if hasattr(obj, 'get') and '/Resources' in obj:
        res = obj['/Resources']
        if hasattr(res, 'get_object'):
            res = res.get_object()

        # 检查字体
        if '/Font' in res:
            fonts = res['/Font']
            if hasattr(fonts, 'get_object'):
                fonts = fonts.get_object()

            for name, ref in fonts.items():
                font = ref.get_object() if hasattr(ref, 'get_object') else ref
                base_font = font.get('/BaseFont', '') if hasattr(font, 'get') else ''
                # 水印字体特征：AAAAAB+Helvetica-Bold
                if 'AAAAAB+Helvetica-Bold' in str(base_font):
                    return True

        # 递归检查嵌套 XObject
        if '/XObject' in res:
            xo = res['/XObject']
            if hasattr(xo, 'get_object'):
                xo = xo.get_object()
            for name, ref in xo.items():
                inner = ref.get_object() if hasattr(ref, 'get_object') else ref
                if check_xobject_has_watermark_font(inner, depth + 1, max_depth):
                    return True

    return False
```

### 移除 Do 命令块

```python
import re
import zlib
from PyPDF2.generic import StreamObject, NameObject, NumberObject

# PyPDF2 返回的 XObject 名称已包含斜杠，如 '/_1'
for xobj_name in watermark_xobject_names:  # xobj_name = '/_1'
    # 匹配格式: q\r\n1 0 0 1 X Y cm\r\n/Name Do\r\nQ
    pattern = rf'q\r\n1 0 0 1 [\d.]+ [\d.]+ cm\r\n{re.escape(xobj_name)}\s+Do\r\nQ'
    text = re.sub(pattern, '', text)
```

### 重建内容流

```python
# 重新压缩并创建新流
new_data = zlib.compress(text.encode('latin-1'))
stream = StreamObject()
stream._data = new_data
stream[NameObject('/Filter')] = NameObject('/FlateDecode')
stream[NameObject('/Length')] = NumberObject(len(new_data))  # 必须用 NumberObject
```

## 常见陷阱

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 正则不匹配 | XObject 名称已含斜杠 | 直接使用 `xobj_name`，不要再加 `/` |
| ValueError: value must be PdfObject | 直接用整数设 Length | 用 `NumberObject(len)` 包装 |
| 内容损坏 | 编码错误 | 用 `latin-1` 编码，设置 `errors='ignore'` |
| 水印未移除 | 嵌套 XObject | 递归检查，设置合适的 max_depth |

## 验证方法

```python
import pdfplumber

# 检查水印字符是否已移除
with pdfplumber.open(output_path) as pdf:
    page = pdf.pages[0]
    watermark_chars = [c for c in page.chars if 'AAAAAB' in c.get('fontname', '')]
    print(f"剩余水印字符: {len(watermark_chars)}")
```

## 依赖

```bash
pip install PyPDF2 pdfplumber
```

- **PyPDF2**: 读写 PDF，操作内部结构
- **pdfplumber**: 验证结果，分析字符
