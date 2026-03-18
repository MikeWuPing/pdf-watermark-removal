# PDF Watermark Removal Skill

一个用于 Claude Code 的 Skill，通过分析 PDF 内部结构识别并移除水印，同时保持正常内容完整。

## 功能特性

- **智能检测**：递归检查嵌套 XObject，识别水印特征字体
- **精准移除**：从内容流中删除 Do 命令块，不影响正常内容
- **完整保留**：100% 保留原文档内容、格式和结构
- **批量处理**：支持处理多页、多文件

## 安装

将 `SKILL.md` 放置到 Claude Code 的 skills 目录：

```bash
# Windows
C:\Users\<用户名>\.claude\skills\pdf-watermark-removal\SKILL.md

# macOS/Linux
~/.claude/skills/pdf-watermark-removal/SKILL.md
```

## 依赖

```bash
pip install PyPDF2 pdfplumber
```

| 库         | 用途                   |
| ---------- | ---------------------- |
| PyPDF2     | 读写 PDF，操作内部结构 |
| pdfplumber | 验证结果，分析字符     |

## 使用方法

在 Claude Code 中直接描述任务：

```
帮我移除 input/ 目录下 PDF 文件的水印
```

Claude 会自动加载此 Skill 并按照指南执行。

## 技术原理

### 水印结构

```
Page → Resources → XObject → 水印XObject → 包含水印字体
                 ↘ Contents → Do命令 → 绘制水印XObject
```

水印通常存在于 Form XObject 中，页面内容流通过 `q ... cm /Name Do Q` 命令块绘制。

### 移除流程

1. **分析**：检查 XObject 是否包含水印字体（如 `AAAAAB+Helvetica-Bold`）
2. **定位**：记录水印 XObject 的名称
3. **移除**：从内容流中删除对应的 Do 命令块

## 示例效果

| PDF 文件                                     | 原始水印字符 | 处理后 |
| -------------------------------------------- | ------------ | ------ |
| 730955_ARL_S_SysMem_OC_TWP_Rev0p9.pdf        | 3281         | 0      |
| 813502_ARL_Overlock_Seminar_TA_WW46_2024.pdf | 2236         | 0      |

## 核心代码

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

### 移除 Do 命令块

```python
import re

# PyPDF2 返回的 XObject 名称已包含斜杠，如 '/_1'
for xobj_name in watermark_xobject_names:
    pattern = rf'q\r\n1 0 0 1 [\d.]+ [\d.]+ cm\r\n{re.escape(xobj_name)}\s+Do\r\nQ'
    text = re.sub(pattern, '', text)
```

## 常见陷阱

| 问题       | 原因                 | 解决方案                               |
| ---------- | -------------------- | -------------------------------------- |
| 正则不匹配 | XObject 名称已含斜杠 | 直接使用 `xobj_name`，不要再加 `/` |
| ValueError | 直接用整数设 Length  | 用 `NumberObject(len)` 包装          |
| 内容损坏   | 编码错误             | 用 `latin-1` 编码                    |
| 水印未移除 | 嵌套 XObject         | 递归检查，增大 max_depth               |

## 适用场景

- PDF 文档包含需要移除的水印
- 水印使用特定字体（如 `AAAAAB+Helvetica-Bold`）
- 批量处理 PDF 水印移除

## 不适用场景

- 扫描件水印（图像层）
- 加密 PDF

## License

MIT
