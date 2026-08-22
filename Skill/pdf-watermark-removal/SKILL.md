---
name: pdf-watermark-removal
description: Use when PDF contains watermarks that need removal — covers XObject-embedded watermarks drawn via Do commands and content-stream text watermarks (text drawn directly in the page stream with a distinctive embedded subset font), or when asked to clean/remove watermarks from PDF documents. Not for scanned image watermarks or encrypted PDFs.
---

# PDF 水印移除

## 概述

水印在 PDF 内部有两种常见形态：**XObject 型**（水印是 Form XObject，页面内容流通过 `Do` 命令绘制）和**内容流文本型**（水印文本用特征字体直接写在页面内容流中）。两者共性的可识别特征：**每页重复出现、且使用独立于正文的嵌入子集字体**（BaseFont 常有 `AAAA`/`AAAAAA` 等子集前缀，如 `AAAAAB+Helvetica-Bold`）。移除原则：定位水印字体 → 找到文字绘制的作用域 → 仅删除该作用域，正文内容流保持原样。

## 识别类型（第一步，必做）

误判类型是"报告无水印但水印还在"最常见的原因。先统计每页字体分布：

```python
from collections import Counter
with pdfplumber.open(pdf_path) as pdf:
    for i, page in enumerate(pdf.pages[:3]):
        print(f'第{i+1}页:', dict(Counter(c['fontname'] for c in page.chars)))
```

水印字体特征：每页**字符数恒定** × 使用**非正文的嵌入子集字体**。

| 类型 | 内部特征 | 水印字体位置 | 移除点 |
|------|----------|-------------|--------|
| XObject 型 | 水印字体藏在 Form XObject 的 `/Resources` 中 | XObject 资源树 | 页面内容流中的 `q ... /Name Do Q` 块 |
| 文本型 | 水印字体在页面顶级 `/Resources`，`BT...ET` 直接绘制 | 页面资源 | 内容流中含特征字体 `Tf` 的最外层 `q ... Q` 段 |

扫描件水印（图像层）和加密 PDF 不适用本方法。

## XObject 型移除

1. 递归扫描 XObject 树，找包含水印字体的叶子（Form XObject 内 `/Font` 有特征 BaseFont），深度上限 20
2. 追溯引用链，找到引用水印叶子的**页面级** XObject 名（如 `/_0`）
3. 从页面内容流删除 `q ... cm /Name Do Q` 块：

```python
import re
# XObject 名已含斜杠，直接使用，勿再加 /
pattern = rf'(?:Q\r\n)?q\r\n(?:[\d.\-]+ ){{5}}[\d.\-]+ cm\r\n{re.escape(xobj_name)}\s+Do\r\nQ'
```

4. 注意 XObject 名成对出现（如 `_0`/`_1`），检测到任一就同时移除两个

## 文本型移除（核心）

```python
import pikepdf, re

pdf = pikepdf.open(pdf_path)
# 1. 从字体资源找特征字体名（匹配子集前缀或唯一非正文字体）
wm_name = None
for name, ref in pdf.pages[0].Resources['/Font'].items():
    font = ref.get_object() if hasattr(ref, 'get_object') else ref
    if 'AAAA' in str(font.get('/BaseFont', '')):  # 嵌入子集特征
        wm_name = name
assert wm_name, '未找到特征字体'

for page in pdf.pages:
    c = page.Contents
    data = c.read_bytes().decode('latin-1', errors='ignore')
    tfs = [m.start() for m in re.finditer(re.escape(wm_name) + r' [\d.]+ Tf', data)]
    if not tfs:
        continue
    # 2. 定位包围全部水印文本块的最外层 q...Q
    q_pos = [m.start() for m in re.finditer(r'(?<![a-zA-Z0-9])q(?![a-zA-Z0-9])', data)]
    Q_pos = [m.start() for m in re.finditer(r'(?<![a-zA-Z0-9])Q(?![a-zA-Z0-9])', data)]
    start = [p for p in q_pos if p < tfs[0]][-1]
    end = [p for p in Q_pos if p > tfs[-1]][0]
    new_data = data[:start] + data[end+1:]
    # 3. 结构安全断言
    assert new_data.count('BT') == new_data.count('ET')
    assert len(re.findall(r'(?<![a-zA-Z0-9])q(?![a-zA-Z0-9])', new_data)) == \
           len(re.findall(r'(?<![a-zA-Z0-9])Q(?![a-zA-Z0-9])', new_data))
    c.write(new_data.encode('latin-1', errors='ignore'))
pdf.save(output_path)
```

水印块常见外部包裹：`q` + 旋转矩阵（如 `0.86603 0.5 -0.5 0.86603 0 0 cm`）+ 若干 `BT` 文本块 + `Q`，整段删除即可。

## 验证方法

```python
import pdfplumber, fitz

orig = pdfplumber.open(input_path); out = pdfplumber.open(output_path)
wm_o = sum(1 for p in orig.pages for c in p.chars if 'AAAA' in c.get('fontname', ''))
body_o = sum(1 for p in orig.pages for c in p.chars if 'AAAA' not in c.get('fontname', ''))
wm_n = sum(1 for p in out.pages for c in p.chars if 'AAAA' in c.get('fontname', ''))
body_n = sum(1 for p in out.pages for c in p.chars if 'AAAA' not in c.get('fontname', ''))
print(f'水印字符: {wm_o} -> {wm_n}')   # 应归零
print(f'正文字符: {body_o} -> {body_n}')  # 应完全一致
# 渲染确认无损坏
fitz.open(output_path)[0].get_pixmap(dpi=80)
```

## 常见陷阱

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 报告无水印但水印可见 | 误判类型：XObject 检测扫不到文本型 | 先做字体分布分析再选分支 |
| 正则匹配失败 | 资源名含 `+`/`.` 等元字符 | `re.escape(资源名)` |
| 内容流乱码/损坏 | 非 ASCII 字符 | 读写统一用 `latin-1` + `errors='ignore'` |
| 删除后 PDF 损坏 | `q/Q` 或 `BT/ET` 不配对 | 删除后断言配对平衡 |
| 保存 PermissionError | 输出文件带只读属性（`shutil.copy2` 遗留） | 先删除旧文件再 `pdf.save` |
| 水印嵌套多层 XObject | 引用链长 | 递归检查，max_depth 到 20；用内容流含叶子名做二次验证 |
| `/F2+0` 类资源名匹配困难 | 内容流里有 `+0` 后缀区分水印字体 | 精确匹配字体资源名，勿宽泛匹配 `/F2` |

## 依赖

```bash
pip install pikepdf pdfplumber PyMuPDF
```

- **pikepdf**：结构操作与内容流读写（`stream.write()` 直接改写，无需手工重建流）
- **pdfplumber**：字体分布分析、字符级验证
- **PyMuPDF (fitz)**：渲染完整性检查
