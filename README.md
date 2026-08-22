# PDF Watermark Removal Skill

一个用于 Claude Code 和 OpenClaw 的 Skill，通过分析 PDF 内部结构识别并移除水印，同时保持正常内容完整。支持两种水印形态：**XObject 型**（水印打包为 Form XObject）与**内容流文本型**（水印文本直接写在页面内容流中）。

## 功能特性

- **智能检测**：先做字体分布分析识别水印类型——XObject 型递归扫描嵌套 XObject 树，文本型定位独立于正文的嵌入子集字体
- **精准移除**：XObject 型删除内容流中的 Do 命令块；文本型删除覆盖全部水印文本块的最外层 `q ... Q` 段（无包裹时降级删 `BT ... ET` 块）
- **完整保留**：100% 保留原文档内容、格式和结构，删除后校验 `q/Q` 与 `BT/ET` 配对平衡
- **批量处理**：支持处理多页、多文件

## 安装

Claude Code Skill的安装方法，OpenClaw只是目录不同，方法类似：将 `SKILL.md` 放置到Claude Code 的 skills 目录：

```bash
# Windows
C:\Users\<用户名>\.claude\skills\pdf-watermark-removal\SKILL.md

# macOS/Linux
~/.claude/skills/pdf-watermark-removal/SKILL.md
```

## 依赖

```bash
pip install pikepdf pdfplumber PyMuPDF
```

| 库         | 用途                       |
| ---------- | -------------------------- |
| pikepdf    | 读取 PDF 结构，改写内容流 |
| pdfplumber | 字体分布分析、字符级验证   |
| PyMuPDF    | 渲染完整性检查             |

## 使用方法

在 Claude Code 中直接描述任务：

```
帮我移除 input/ 目录下 PDF 文件的水印
```

Claude 会自动加载此 Skill 并按照指南执行。

## 技术原理

### 水印结构

PDF 水印有两种合法形态，检测逻辑必须分别覆盖：

```
形态 A（XObject 型）：
Page → Resources → XObject → 水印XObject → 包含水印字体
                 ↘ Contents → Do命令 → 绘制水印XObject

形态 B（内容流文本型）：
Page → Resources → /Font（特征子集字体）
    → Contents → BT ... /Fontname Tf ... Tj ... ET（文本直接绘制）
```

两种形态的水印都具有可识别特征：每页重复出现、使用独立于正文的子集字体（BaseFont 常以 `AAAA` 前缀开头，如 `AAAAAB+Helvetica-Bold`）。

### 识别流程

1. **字体分布分析**：统计每页字符的字体分布，字符数恒定且非正文使用的子集字体 → 文本型水印
2. **XObject 递归扫描**：嵌套结构中发现特征字体 → XObject 型水印
3. 两者都不命中时，才如实报告"没有水印"——不把"只见了 XObject 树"当成"整个 PDF 没有水印"

### 移除流程

XObject 型：分析 → 追溯页面级 XObject 名称 → 从内容流中删除 `q ... cm /Name Do Q` 命令块

内容流文本型：扫描内容流定位特征字体的所有 `Tf` 命令 → 找到**覆盖全部水印文本块的最外层 `q ... Q` 段** → 整段删除（旋转矩阵、颜色、透明度等状态随段一并移除，不留空壳状态）；无 `q ... Q` 包裹时降级为逐块删除 `BT ... ET`

## 核心代码

### 识别水印类型（先分析，再动手）

```python
import pdfplumber
from collections import Counter

with pdfplumber.open(pdf_path) as pdf:
    for page in pdf.pages[:3]:
        print(dict(Counter(c['fontname'] for c in page.chars)))
# 每页恒定字符数 + 非正文子集字体（AAAA 前缀）=> 内容流文本型
```

### XObject 型：递归检测水印字体

```python
def check_xobject_has_watermark_font(obj, depth=0, max_depth=10):
    """递归检查 XObject 是否包含水印字体"""
    if depth > max_depth:
        return False
    if hasattr(obj, 'get') and '/Resources' in obj:
        res = obj['/Resources']
        if hasattr(res, 'get_object'):
            res = res.get_object()
        if '/Font' in res:
            fonts = res['/Font']
            if hasattr(fonts, 'get_object'):
                fonts = fonts.get_object()
            for name, ref in fonts.items():
                font = ref.get_object() if hasattr(ref, 'get_object') else ref
                base_font = font.get('/BaseFont', '') if hasattr(font, 'get') else ''
                if 'AAAAAB+Helvetica-Bold' in str(base_font):
                    return True
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

### 文本型：定位并删除最外层 q...Q 段

```python
import pikepdf, re

pdf = pikepdf.open(pdf_path)
# 1. 从页面资源中找特征字体名
wm_name = next((name for name, ref in pdf.pages[0].Resources['/Font'].items()
                if 'AAAA' in str(ref.get_object().get('/BaseFont', ''))), None)

for page in pdf.pages:
    data = page.Contents.read_bytes().decode('latin-1', errors='ignore')
    tfs = [m.start() for m in re.finditer(re.escape(wm_name) + r' [\d.]+ Tf', data)]
    if not tfs:
        continue
    qs = [m.start() for m in re.finditer(r'(?<![a-zA-Z0-9])q(?![a-zA-Z0-9])', data)]
    Qs = [m.start() for m in re.finditer(r'(?<![a-zA-Z0-9])Q(?![a-zA-Z0-9])', data)]
    start = [p for p in qs if p < tfs[0]][-1]
    end = [p for p in Qs if p > tfs[-1]][0]
    new_data = data[:start] + data[end + 1:]
    # 安全断言：配对结构必须保持平衡
    assert new_data.count('BT') == new_data.count('ET')
    page.Contents.write(new_data.encode('latin-1', errors='ignore'))
pdf.save(output_path)
```

## 常见陷阱

| 问题       | 原因                         | 解决方案                               |
| ---------- | ---------------------------- | -------------------------------------- |
| 误判类型   | 只检查 XObject，扫不到文本型 | 先做字体分布分析再选分支               |
| 水印未移除 | 嵌套 XObject                 | 递归检查，max_depth 到 20              |
| 正则不匹配 | 资源名含 `+` 等元字符        | `re.escape(资源名)`                    |
| 内容损坏   | 编码错误                     | 用 `latin-1` 编码，设置 `errors='ignore'` |
| 删除后损坏 | q/Q、BT/ET 不配对            | 删除后断言配对平衡                     |
| 保存报权限错误 | 输出文件带只读属性        | 先删除旧文件再 `pdf.save`              |
| XObject 名成对 | 页面级名称 `_0`/`_1` 并存 | 检测到任一即同时移除两个               |

## 适用场景

- PDF 文档包含需要移除的水印（XObject 型或内容流文本型）
- 水印使用特征字体（如 `AAAAAB+Helvetica-Bold`、`AAAAAA+FontName-0`）
- 批量处理 PDF 水印移除

## 不适用场景

- 扫描件水印（图像层）
- 加密 PDF

## License

MIT
