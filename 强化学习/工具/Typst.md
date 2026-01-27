你截图里这种并不是 Markdown+LaTeX，而是 **Typst**（文件后缀 `.typ`）。
 别人说的“把 PDF 转成这样，然后就能更新 PDF”，通常指的是：

- **不是把 PDF 直接“无损转换”成 `.typ`**（PDF 是排版后的“结果”，很难还原源码）
- 而是：**拿到原文内容（或从 PDF 提取文本/结构）→ 用 Typst 重写排版 → 以后改 `.typ` 就能一键重新生成 PDF**

你截图里右侧是 Typst 的 **实时预览**：改左边 `.typ`，右边 PDF 预览立刻更新。

下面给你两种最常见的“操作路线”，你按自己情况选。

------

## 路线 A：你本来就有源码（最理想）

如果你有：

- Markdown / Word / LaTeX / 纯文本
- 或者原始笔记

那就直接用 Typst 来排版生成 PDF。

### 1）安装 Typst

- Windows 最省事（任选一种）：
  - **Scoop**：`scoop install typst`
  - **Chocolatey**：`choco install typst`
  - 或去官网/Release 下载 `typst.exe` 放到 PATH

装好后终端能跑：

```
typst --version
```

### 2）创建一个 `main.typ`

最小模板（能生成 PDF）：

```
#set page(width: 210mm, height: 297mm, margin: 20mm)
#set text(font: "Times New Roman", size: 11pt)

= 我的标题
这里是正文。

$ V_\pi(s)=\mathbb{E}_\pi[G_t \mid S_t=s] $
```

### 3）编译 PDF

```
typst compile main.typ
```

会生成 `main.pdf`

### 4）实时预览（你截图这种效果）

```
typst watch main.typ
```

它会监控文件变化，自动重新生成 PDF。

------

## 路线 B：你只有 PDF，想“转成可编辑的 Typst”

这条路要分清：**只能“尽量还原”，做不到完美一键转换**。

### 常见做法是 3 步：

### 1）从 PDF 提取文本

- 如果 PDF 是“可选中文字”的（不是扫描图）：

  - 用 `pdftotext`（poppler 工具）：

    ```
    pdftotext input.pdf output.txt
    ```

- 如果 PDF 是扫描版（图片）：

  - 需要 OCR（识别文字），比如：
    - Adobe OCR
    - ABBYY
    - 或者开源 OCR（效果看 PDF 清晰度）

### 2）用 Typst 重排（关键工作）

把提取的文本粘到 `.typ` 里，按章节标题、公式、列表、图片重新排版。

### 3）以后就能像截图那样持续更新

修改 `.typ` → `typst watch` 自动刷新 PDF。

------

## 你截图里那段代码在干什么？

你图里有这种：

```
#show raw.where(lang: "python"): it => {
  show regex("\\$(.*?)\\$"): re => {
    eval(re.text, mode: "markup")
  }
  it
}
```

它大概是在做一个“骚操作”：

- 让代码块里出现的 `$...$` 也能当数学公式解析（通常 raw/code 里默认不会解析公式）
- 方便写“代码 + 公式混排”的文档

这属于 Typst 的高级自定义样式。

------

## 最实际的问题：你现在是哪种情况？

为了不绕弯，我直接给你对应建议：

### ✅ 你有 Word/Markdown/纯文本

直接走 **路线 A**，最快、最干净。

### ✅ 你只有 PDF（可复制文字）

先 `pdftotext` 提取，再 Typst 重排，工作量中等。

### ✅ 你只有扫描 PDF（不能复制）

需要 OCR，重排工作量较大。